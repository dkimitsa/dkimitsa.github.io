---
layout: post
title: "RoboVM MVP for Windows and Linux"
tags: ["linux windows", 'whats new']
date: 2017-12-12 11:00:00 +0200
---
Three month project is over and I can share the result. I built all missing components to make it able to develop iOS projects using Windows and Linux host.  
First of all **what it can't:**  
<!-- more -->

- There is no any simulator available (deploy to device only);
- There is no Interface Builder alternative available, so UIKit UI can't be developed. But you still can deploy it;
- There is no TextureAtlass compiler, but you can attached third party one by using config;

**What it can with limitations:**
- Codesign is limited to binary only, will not sign any embedded Frameworks (will be added soon);
- Xib/Storyboard compiler is full of bugs and can produce only simple UIs (work in progress);
- pngcrunch functionality not implemented as not required for MVP;

**How this was done:**  
There will a technical post with detailed explanations, in two words following was done:
- RoboVM adapted to be able use different approaches depending on OS;
- Compiled libLLVM and LibMobileDevice;
- Implemented code sign;
- Collected toolchain utils replacement for MacOSX ones (linker, ibtool, actool, otool, lipo, dsymutil, strip etc);

**Dependencies:**  
Windows: [iTunes](https://www.apple.com/lae/itunes/download/) has to be installed to provide usbmux layer

**Installation:**  
1. Download and install [MVP RoboVM Idea plugin](https://goo.gl/WxVuM3). Once started you should see following screen:
![]({{ "/assets/2017/12/07/robovm_setup_dialog.png" | absolute_url}})
2. Install toolchain:
   - depending on operation system and arch from screenshot above pick package to download.  
   [windows-x86_64](https://goo.gl/wbz5WJ)  
   [windows-x86 /not tested/](https://goo.gl/zqM4Lg)  
   [linux-x86_64](https://goo.gl/MU7tMW)  
   [linux-x86 /not tested/](https://goo.gl/TemZDA)  
   - change dir to `~/.robovm/platform/` (create if missing) and unpack package.
3. **UPDATE: there is [homemade SDK available]({{ site.baseurl }}{% post_url 2018-01-24-homemade-ios-sdk-for-linux-windows-robovm %}).**  Provide Xcode files. .tbl and device images are needed to run code. Its not allowed to distribute these files due copyright moments.
   - But you should be able to make a backup from your legal copy of Xcode with following script:
```bash
rsync -avmL --include '*/' \
    --include='Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/**/*.tbd' \
    --include='/*.plist' \
    --include='Developer/Platforms/iPhoneOS.platform/*.plist' \
    --include='Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/*.plist' \
    --include='Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk/System/Library/CoreServices/*.plist' \
    --include='Developer/Platforms/iPhoneOS.platform/DeviceSupport/10*/*' \
    --include='Developer/Platforms/iPhoneOS.platform/DeviceSupport/11*/*' \
    --exclude='*' /Applications/Xcode.app/Contents/ Xcode.app   && zip -r xcode_app Xcode.app
```
   - copy backup `xcode_app.zip` to target platform, change dir to `~/.robovm/platform/` and unpack `xcode_app.zip`.
4. At this moment there should be no red lines about xcode/toolchain and installation could be completed.

5. Setup Provision profiles and signing certificates  
   - Provision profiles shall be copied to `~/.robovm/platform/mobileprovision/` (create folder if required)
   - Certificates and private key shall be exported from keychain to .p12 file without password as described [here](https://developer.apple.com/library/content/documentation/IDEs/Conceptual/AppDistributionGuide/MaintainingCertificates/MaintainingCertificates.html#//apple_ref/doc/uid/TP40012582-CH31-SW15) and [here, keychain](https://www.raywenderlich.com/3443/apple-push-notification-services-tutorial-for-ios-part-12/keychain-access-3-export-private-key). Put p12 files to `~/.robovm/platform/keychain/` (create folder if required)  
   - Once these folders are set up provision profiles/certificates shall appear in "Run/Debug configuration"

At the end `~/.robovm/platform` folder shall contain following structure (`windows-x86_64` as example here):
```
C:\USERS\DKIMITSA\.ROBOVM\PLATFORM
+---keychain
|       robovm4windows.p12
+---mobileprovision
|       robovm4windows.mobileprovision
+---windows-x86_64
|       actool.exe
|       arm-apple-darwin11-codesign_allocate.exe
|        ...
|       tapi.dll
|       xib2nib.exe
\---Xcode.app
    |   Info.plist
    |   version.plist
    \---Developer
        \---Platforms
            \---iPhoneOS.platform
                |   Info.plist
                |   ...
                \---DeviceSupport
                    ...
```

**Run/Debug project same way as on MacOSX.**

**Credits**
- [Microsoft/WinObjC for xib2nib](https://github.com/Microsoft/WinObjC)
- [facebook/xcbuild for actool](https://github.com/facebook/xcbuild)
- [tpoechtrager/cctools-port for ld64](https://github.com/tpoechtrager/cctools-port)
