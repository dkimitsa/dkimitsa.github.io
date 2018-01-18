---
layout: post
title: 'Tutorial: using app-extensions in RoboVM iOS application'
tags: ['app-extensions', Tutorial]
---
Apple added support for [app-extensions](https://developer.apple.com/library/content/documentation/General/Conceptual/ExtensibilityPG/index.html) while ago (ios8). I've created today a [PR255](https://github.com/MobiVM/robovm/pull/255) that add ability to include pre-build app extension into RoboVM app. Answering the possible question:
> Can I use RoboVM to develop appext  

*Short answer*: no, not today  
<!-- more -->
*Long answer*: Any RoboVM binary (app, app-extensions, framework etc) will include RoboVM runtime and everything that comes together. This means that both app and extension will be quite big. More over it is not clear enough if it is possible to run two instances of VM. In theory to do this app has to be organized as [Embedded framework]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %}).  Application and app-extensions shall be native wrapper that use this framework. Not good idea.

# Building native app-extension for [OneSignal robopod]({{ site.baseurl }}{% post_url 2018-01-05-robopods-onesignal-framework %})
Official documentation for OneSignal integration is available by [link](https://documentation.onesignal.com/docs/ios-sdk-setup). It will require OneSignal.framework itself, easiest way to get it is to build using [Carthage](https://github.com/Carthage/Carthage#installing-carthage):
```
echo 'github "OneSignal/OneSignal-iOS-SDK"' >> Cartfile
carthage update
```

## First issue: there is no project type `**** Service Extension` in XCode

Service extension can be added to existing project only as target. So lets create new "Single View App" project in xcode. Fill the options dialog with values as shown on screenshot below:  
![]({{ "/assets/2018/01/18/appex-xcode-project-settings.png" | absolute_url}})

Add new `Notification Service Extension` target (File-New target) to newly created project. Enter the product name as `OneSignalNotificationServiceExtension` and press Finish. Do not activate "OneSignalNotificationServiceExtension" scheme.
At this moment project should have following structure:  
![]({{ "/assets/2018/01/18/appex-xcode-project-structure.png" | absolute_url}})

There is no need in "Single View App" so it can be freely removed:
* select OneSignal target and hit '-'
* select OneSignal folder in project tree and hit 'delete'. Select move to trash.

## Adding OneSignal.framework
* Copy `OneSignal.framework` to project directory next to `OneSignalNotificationServiceExtension` folder
* use finder and drag-and-drop `OneSignal.framework` to project tree;
* check that `Build Phases - Link Binary With Libraries` contain `OneSignal.framework`

![]({{ "/assets/2018/01/18/appex-xcode-link-settings.png" | absolute_url}})

## Adding OneSignal code
Just following [manual](https://documentation.onesignal.com/docs/ios-sdk-setup) and turning `NotificationService.m` into following:  
```objc
#import <OneSignal/OneSignal.h>
#import "NotificationService.h"

@interface NotificationService ()
@property (nonatomic, strong) void (^contentHandler)(UNNotificationContent * contentToDeliver);
@property (nonatomic, strong) UNNotificationRequest * receivedRequest;
@property (nonatomic, strong) UNMutableNotificationContent * bestAttemptContent;
@end

@implementation NotificationService
- (void)didReceiveNotificationRequest:(UNNotificationRequest * )request withContentHandler:(void (^)(UNNotificationContent * ) ) contentHandler {
    self.receivedRequest = request;
    self.contentHandler = contentHandler;
    self.bestAttemptContent = [request.content mutableCopy];

    [OneSignal didReceiveNotificationExtensionRequest:self.receivedRequest withMutableNotificationContent:self.bestAttemptContent];

    // DEBUGGING: Uncomment the 2 lines below and comment out the one above to ensure this extension is excuting
    //            Note, this extension only runs when mutable-content is set
    //            Setting an attachment or action buttons automatically adds this
    // NSLog(@"Running NotificationServiceExtension");
    // self.bestAttemptContent.body = [@"[Modified] " stringByAppendingString:self.bestAttemptContent.body];

    self.contentHandler(self.bestAttemptContent);
}

- (void)serviceExtensionTimeWillExpire {
    // Called just before the extension will be terminated by the system.
    // Use this as an opportunity to deliver your "best attempt" at modified content, otherwise the original push payload will be used.

    [OneSignal serviceExtensionTimeWillExpireRequest:self.receivedRequest withMutableNotificationContent:self.bestAttemptContent];

    self.contentHandler(self.bestAttemptContent);
}
@end
```

At this moment build should work (Product/Build)

## Adding signature
Configure target's settings `General-signing` with any certificate and provisioning profile that matches extension bundle id (wildcard provisioning profile rules here) or just [create and setup dummy certificate/profile]({{ site.baseurl }}{% post_url 2018-01-04-ios-homemade-provision-profile %}) for signing.  
In general signature doesn't matter here as when included to RoboVM project app extension will be resigned. It is only required to build binary with `xcodebuild`

## Building universal binary
To create `fat` binary simulator and arm slices are being build separately and then combined using `lipo`. It is being done by invoking following commands in ternimal. Just make sure to do it in folder whene `.xcodeproj` is located:   
```bash
xcodebuild -project onesignal.xcodeproj -target OneSignalNotificationServiceExtension -configuration release -sdk iphoneos -arch arm64 -arch armv7 -arch armv7s BUILD_DIR=build BUILD_ROOT=build
xcodebuild -project onesignal.xcodeproj -target OneSignalNotificationServiceExtension -configuration release -sdk iphonesimulator -arch i386 -arch x86_64 BUILD_DIR=build BUILD_ROOT=build

mkdir "OneSignalNotificationServiceExtension.appex"
lipo -create -output "OneSignalNotificationServiceExtension.appex/OneSignalNotificationServiceExtension" \
    "build/release-iphoneos/OneSignalNotificationServiceExtension.appex/OneSignalNotificationServiceExtension" \
    "build/release-iphonesimulator/OneSignalNotificationServiceExtension.appex/OneSignalNotificationServiceExtension"

cp "build/release-iphoneos/OneSignalNotificationServiceExtension.appex/Info.plist" "OneSignalNotificationServiceExtension.appex/"

# resign with empty signature
codesign -f -s - "OneSignalNotificationServiceExtension.appex"
```

Application extention is ready to be used with RoboVM

## Adding to RoboVM project
[PR255](https://github.com/MobiVM/robovm/pull/255) have added following configuration key to `robovm.xml`:
- `<appExtensionPaths>`, type `<path>`: specifies a list of directories where to look for app ext. Works similar to `frameworkPaths`;
- `<appExtensions>`, type `<extension>`: list extensions to include in app.  

Example:  
```xml
<appExtensionPaths>
    <path>libs</path>
</appExtensionPaths>
<appExtensions>
    <extension>OneSignalNotificationServiceExtension</extension>
</appExtensions>
```

## What RoboVM does to application extensions
1. During `doInstall` phase it copies appex to `PlugIns` folder and updates `CFBundleIdentifier` in `Info.plist` as it has extends application ones;
2. During `prepareInstall` phase it resigns appex with signature used for application

## Complete sample available at [github](https://github.com/dkimitsa/robovm-samples/tree/alt/robopods/onesignal)
