---
layout: post
title: 'Support for swift dependencies for static libs/frameworks'
tags: [fix, debugger, idea]
---
RoboVm compiler is able to detect swift usage in dynamic frameworks and copies all required swift libraries into application bundle.  
Same `logic` (calling otool -L) doesn't work for static libraries (.a) and static frameworks.  
While `xcframework` and better logic (instead of `otool -L`) is still in development workaround was introduced to unblock community:
> swift libraries have to be manually listed in `robovm.xml`:

Example: 
```xml
<libs>
    <lib>libswiftCore.dylib</lib>
    <lib>libswiftCore.dylib</lib>
    <lib>libswiftCoreFoundation.dylib</lib>
    <lib>libswiftCoreGraphics.dylib</lib>
    <lib>libswiftCoreImage.dylib</lib>
    <lib>libswiftDarwin.dylib</lib>
    <lib>libswiftDispatch.dylib</lib>
    <lib>libswiftFoundation.dylib</lib>
    <lib>libswiftMetal.dylib</lib>
    <lib>libswiftObjectiveC.dylib</lib>
    <lib>libswiftQuartzCore.dylib</lib>
    <lib>libswiftUIKit.dylib</lib>
</libs>
```

Simplest way to find out list of required swift libraries is to use `otool -l libOne.a | grep "\-lswift"`.

Now compiler detects swift library in dependencies and applies logic similar to dynamic frameworks:  
- adds swift library path to library search path;
- link binary with these libraries; 
- copies swift libraries into application bundle;

Code was delivered as [PR474](https://github.com/MobiVM/robovm/pull/474).
