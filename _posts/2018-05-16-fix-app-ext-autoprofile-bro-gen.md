---
layout: post
title: 'Maintenance: bugs in AppExtensions, Auto signature/profile selection and bro-gen'
tags: [fix, 'app-extensions', 'bro-gen']
---
Today bug-fix/maintenance delivers following:  

1. Fixed automatic profile/signature selection as it was picking first random signature with `iPhone Developer|iOS Development` prefix;
2. Improved [app-extension support]({{ site.baseurl }}{% post_url 2018-01-18-tutorial-using-app-extensions-with-robovm %}): apple was rejecting binaries due missing provisioning profile and entitlements.
3. Small improvements to `bro-gen` that can now pick string macro into constants

Details:  
<!-- more -->
## Issue with automatic profile/signature selection
Old code was broken due following: if signature was set to 'Auto' it was picking any `iPhone Developer|iOS Development` one without any respect to provisioning profile. This caused following use cases broken:
1. `auto signature` and `manual profile`: randomly picked signature will not match profile (in case you are not lucky and have lot of signatures/profiles);
2. `auto signature` and `auto profile` will not work as well as signature is random;

Fix provides following:
1. `auto signature` and `manual profile`: picks only signatures that matches profile;
2. `auto signature` and `auto profile`: first set of profiles that match bundle id is fetched. This set is being intersected with list of identities to find a profile and signature pair that matches each other.  
[PR293](https://github.com/MobiVM/robovm/pull/293), [commit1](https://github.com/dkimitsa/robovm/commit/531aa4c87698aa3bc137a402eeecb56d8c88b2fa), [commit2](https://github.com/dkimitsa/robovm/commit/ea70289143497beab63f2980f86d5e4dc44ff990)


## Issue with app-extensions
As [reported by github user @sofroma](https://gitter.im/MobiVM/robovm?at=5af579e85e8e2175e259995a) there were issues while submiting application to appstore that contains extensions:
* Missing Code Signing Entitlements. No entitlements found in bundle 'xxx.yyy.zzz.StickerPackExtension'
* Missing Provisioning Profile in bundle 'xxx.yyy.zzz.StickerPackExtension'

RoboVM was extended to copy provisioning profile and generate entitlement for app-extension. Provisioning profile for app-extension is `must have` now. Requirement for profile is following:  
1. It shall contain same `signing identity` as profile of main application;
2. It's application id shall match id of application extension. RoboVM makes bundle id for it as `MainAppId + "." + ExtensionName`. For `OneSignalNotificationServiceExtension` it results in `com.sample.application.OneSignalNotificationServiceExtension`, here `com.sample.application` is bundle id of applications, so bundle id of provision profile shall be:
  * either exact match to bundle Id RoboVM generates for app extension (e.g. `com.sample.application.OneSignalNotificationServiceExtension`)
  * wildcard id (e.g. * ) which is preferable way as it allows to have one profile for many extensions.

 RoboVM will automatically search for profile that matches the signature and extension bundle id. Also there is an option to explicitly specify it in `robovm.xml` with `profile` parameter to `extension` entry:
 ```xml
 <appExtensions>
     <extension profile="3AED05A9-5F1F-4120-9276-11980B9C88EE">OneSignalNotificationServiceExtension</extension>
 </appExtensions>
 ```

Value of profile is same `iosProvisioningProfile` used with gradle and can have following values:
- either `udid` of profile;
- either `profile name`;
- either `appIdPrefix`;
- either `appIdName`.

Application extension tutorial [was updated]({{ site.baseurl }}{% post_url 2018-01-18-tutorial-using-app-extensions-with-robovm %}) as well.  
[PR293](https://github.com/MobiVM/robovm/pull/293), [commit](https://github.com/dkimitsa/robovm/commit/ea70289143497beab63f2980f86d5e4dc44ff990)

## Improvement of `bro-gen`
[@irina88 asked question](https://gitter.im/MobiVM/robovm?at=5afb2fe32df44c2d062db775) why `bro-gen` doesn't handle string macro automatically (like `#define AFEventLevelAchieved @"af_level_achieved"`). bro-gen was converting only numbers in macro into constants, small tweak allows simple string macro to be converted into string constants, check [this commit](https://github.com/dkimitsa/robovm-bro-gen/commit/c9806dfbda7604610bb0b0666b151b2d3b981c6a).  
Now configure yaml for constant capture:
```yaml
constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    (.*):
        class: __FixMe
        name: 'Constant__#{g[0]}'
```
it will produce code as bellow:
```
 /*<constants>*/
 public static final String Constant__AFEventLevelAchieved = "af_level_achieved";
 /*</constants>*/
 ```
