---
layout: post
title: 'ClassCastException: ObjC protocol and its Java implementation'
tags: ['fix']
---

## Background:
There is many API in CocoaTouch where argument is ObjC protocol type. In RoboVM protocols are mapped into `interfaces`. The problems begin when bro-bridge tries to marshall this protocol implementation into the native object:  
```java
UIToolbar toolbar = new UIToolbar();
toolbar.setDelegate(bar -> UIBarPosition.Any);
```
causes: 
```
java.lang.ClassCastException: com.mycompany.myapp.Main$$Lambda$1 cannot be cast to org.robovm.objc.ObjCObject
	at org.robovm.objc.ObjCObject$Marshaler.protocolToNative(ObjCObject.java:388)
	at org.robovm.apple.uikit.UIToolbar.$m$setDelegate$(Native Method)
	at org.robovm.apple.uikit.UIToolbar.setDelegate(UIToolbar.java)
	at com.mycompany.myapp.Main.didFinishLaunching(Main.java:26)
```

Root case: protocol marshaller can handle only ObjCObject. Indeed, there is no way to marshal random Java class to ObjC native world.  
## The existing workaround:
<!-- more -->  
Simple case would be:
- use NSObject subclass that implements interface;
- use Adapter class if it is available; 

In any case this overcomplicates the code, new coders still will face this issue and usage of simple lambda is not allowed. 

## The fix
Idea is to force classes that implement `NSObjectProtocol` and extends from `java.lang.Object` to extend from `NSObject`.  
RoboVM compiler was extended with `ObjCProtocolToObjCObjectPlugin` that responsible for:
- changing super class to `NSObject`;
- modifying initializer (constructor) code to call `NSObject.<init>()` instead of `Object.<init>()`


## Code 
Changes proposed as [PR563](https://github.com/MobiVM/robovm/pull/563)
