---
layout: post
title: 'Why enhance.co injected functionlity does not work mostly in RoboVM (as per today)'
tags: [bug, hacking, robopods]
---

User OceanBreezeGames reported following issue on [Gitter channel](https://gitter.im/MobiVM/robovm?at=5bd9dfa5538a1c19715143ee) that was interesting to investigate:
```
Enhanced ipa fails with:
Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '+[Enhance isInterstitialReady]: unrecognized selector sent to class 0x103343eb8'
```

Tried enhancing the app several time and it sometimes worked and sometimes not. For start how it expected to work:
<!-- more -->
`enhance.co` provides simple api `libconnector.a` to be used during development with simplified advertisement/push etc api. And once app is ready it to be sent to them for SDK injection and other magic.  
Basically crash reported due exception around very primary interface of connector.

## Why `isInterstitialReady` is unrecognized
After injection `ipa` file is extended with `enhance.framework`. And connector is expected to work with it.
Quick check has shown that both `enhance.framework` and `libconnector.a` has `Enhance` class.

Reason: classical name space conflict.

## Quick check that wrong class is used
Running following snippet:
```java
    // 1
    NSBundle n = NSBundle.getBundle(Enhance.class);
    System.out.println(n.getBundleURL());
    // 2
    ObjCClass c = ObjCClass.getByName("Enhance");
    System.out.println(c.toDebugString());
    // 3
    try {
        System.out.println(NSBundle.getMainBundle().getClassNamed("Enhance"));
    } catch (Exception e) {
        e.printStackTrace();
    }
```
shows expected result:
```
// 1:
file:///var/containers/Bundle/Application/E47E3BDA-9356-4C28-973F-199EE5655BD3/Main.app/Frameworks/enhance.framework/
// 2:
@interface Enhance : NSObject <WaterfallClientDelegate>
@end
// 3:
org.robovm.objc.ObjCClassNotFoundException: Could not find Java class corresponding to Objective-C class: nil
	at org.robovm.objc.ObjCClass.toObjCClass(ObjCClass.java:320)
	at org.robovm.objc.ObjCClass$Marshaler.toObject(ObjCClass.java:105)
```

## Why is this happening
RoboVM uses dynamic linking and doesn't link statically to `Enhance` from `libconnector.a`. And actualy it uses `objc_getClass` during runtime to get class pointer. Sadly at this moment there is already `enhance.framework` loaded that breaks everything.

## Simulating with XCode gives corresponding message
```
objc[1327]: Class Enhance is implemented in both /private/var/containers/Bundle/Application/1B568A78-4EC2-4C9D-A9DD-440038D9BF5D/xcode10_test.app/Frameworks/enhance.framework/enhance (0x1016c2548) and /var/containers/Bundle/Application/1B568A78-4EC2-4C9D-A9DD-440038D9BF5D/xcode10_test.app/xcode10_test (0x1011af290). One of the two will be used. Which one is undefined.
```

## Fix
Simple and proper fix to be done by `enhance.co` is to rename `Enhance` class inside `enhance.framework`. As errors are happening due their hacky way of service.
