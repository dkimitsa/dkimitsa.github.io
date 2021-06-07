---
layout: post
title: 'ios12: class_copyProtocolList crash'
tags: [bug, ios12, swift]
---

Issue was reported on [gitter](https://gitter.im/MobiVM/robovm?at=60b8a6ccbdecf719a09623f9) and caused crash inside `ObjC` library of RoboVM with following stacktrace:  
```
0 _mapStrHash(_NXMapTable*, void const*) + 4
1 _NXMapMember(_NXMapTable*, void const*, void**) + 52
2 NXMapGet + 20
3 getProtocol(char const*) + 28
4 class_copyProtocolList + 328
5 [J]org.robovm.objc.ObjCRuntime.class_copyProtocolList(JJ)J + 4358277632
```

Issue was reproduced on iOS12 simulator. Sample was simplified to following `RoboVM/Java` snippet: 
<!-- more -->
```java
@CustomClass("Test")
public class Test extends NSObject{
    @Method
    public void foo(NSDictionary<NSString, NSString> p) {
        System.out.println("Hello: " + p);
    }
}
```

That to be called from Swift code like bellow:
```swift
  Test().foo([:])
```

As there is a lot of work happening in marshaller code of [ObjCObject.toObjCObject](https://github.com/MobiVM/robovm/blob/90cec2fb20257e814dcf5c1fee283a098742a5b4/compiler/objc/src/main/java/org/robovm/objc/ObjCObject.java#L265) sample was modified to eliminate marshaller code and do it in minimal code:  
```java
    @Method
    public void foo(@Pointer long handle){
        long classPtr=ObjCRuntime.object_getClass(handle);
        long protocols=ObjCRuntime.class_copyProtocolList(classPtr,0);
    }
```

It produces same crash as in top of post. Strange thing if similar code is implemented in `Objective-C` and called from swift WORKS. Both `RoboVM` and `ObjectiveC` cases were debugged with XCode.  
In case of `RoboVM` structures of `cls->data()->protocols` were not initialized and code picked garbage string. Thanks, Apple, for opensourcing this [code](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-runtime-new.mm.auto.html).   

Search for similar cases bring me to [Xamarin](https://github.com/xamarin/xamarin-macios/pull/6293) and root case is bug in `ObjectiveC` runtime.  
The fix is to call `cls = [obj class]` instead of `cls = ObjCRuntime.object_getClass(obj)`.  
In case of `RoboVM` things are bit complicated as code works on `ObjectiveC` level and there is no `NSObject` protocol known. So the way is to use selectors:  
```java
    public static ObjCClass getFromObject(long handle) {
        long classPtr = ObjCRuntime.object_getClass(handle);
        // dkimitsa. There is a bug observed in iOS12 that causes not all Objective-C class fields properly initialized
        // in Class instance of Swift classes. This causes a crash in APIs like class_copyProtocolList to crash
        // as it faces not initialized data. Example of such class is `Swift.__EmptyDictionarySingleton`.
        // Workaround for this case is to call [NSObject class] selector that initializes all structs.
        // ObjCClass is not always NSObject check for responds to selector is required here.
        // similar case: https://github.com/xamarin/xamarin-macios/pull/6293
        if (classPtr != 0 && ObjCRuntime.class_respondsToSelector(classPtr, SELECTOR_NSOBJECT_CLASS.getHandle())) {
            classPtr = ObjCRuntime.ptr_objc_msgSend(handle, SELECTOR_NSOBJECT_CLASS.getHandle());
        }
        return toObjCClass(classPtr);
    }
```

The fix was delivered as [PR588](https://github.com/MobiVM/robovm/pull/588)
