---
layout: post
title: 'Debugger: "Could not initialize class" when running debug build'
tags: [debugger, bug, rollback]
---

There was a report in `MobiVM` matrix channel for crash that is happening only during running under debugger:
```
java.lang.NoClassDefFoundError: Could not initialize class com.sample.mobile.ios.network.RoboVmNetzwerkStatusController
        at com.sample.mobile.ios.Main.didFinishLaunching(Main.java:208)
        at com.sample.mobile.ios.Main.$cb$application$didFinishLaunchingWithOptions$(Main.java)
        at org.robovm.apple.uikit.UIApplication.main(Native Method)
        at org.robovm.apple.uikit.UIApplication.main(UIApplication.java:442)
        at com.sample.mobile.ios.Main.main(Main.java:314)
```


# TLTR: Fix and workaround
<!-- more -->

- Rollback of [#754](https://github.com/MobiVM/robovm/pull/754);
- Call `Class.forName("org.robovm.objc.ObjCObject");` somewhere early in iOS code;

# Background
it was happening in production-wide code base but luckily its was simplified to 10 file mini project what was reduced to following sample:

```java
package com.example.mobile.ios;

import org.robovm.apple.foundation.NSObject;
import org.robovm.apple.foundation.NSString;

public class Main {
    public static void main(String[] args) throws Exception {
        new NSObject(); // cause Failed to preload inner ObjCClass host class com.example.mobile.ios.Foo: null
        new Foo();      // cause java.lang.NoClassDefFoundError: Could not initialize class com.example.mobile.ios.Foo
        new Foo.FooInner(); // to keep class from being tree shaked
    }
}

class Foo extends NSObject {
    static NSString npeHere = new NSString("NPE is here");

    public static class FooInner extends NSObject {
        public static void foo() { Foo.npeHere.toString(); }
    }
}
```

Once started it crashes with following output:
```
Failed to preload inner ObjCClass host class com.example.mobile.ios.Foo: null
java.lang.NoClassDefFoundError: Could not initialize class com.example.mobile.ios.Foo
	at com.example.mobile.ios.Main.main(Main.java:9)
```

From these messages it seems that:
- `Failed to preload inner ObjCClass host class` comes from changes added in [#754](https://github.com/MobiVM/robovm/pull/754); 
- `NoClassDefFoundError` is from `new Foo();`

Both are related. As `preload inner ObjCClass host class` actually does `Class.forName("com.example.mobile.ios.Foo");` and seems like it is failed during class initialization. Thus, any future operations with this class will fail. 

# Investigation 
To get insights apps was started int verbose/trace mode by specifying `-rvm:log=trace` VM argument, in case of Simulator invocation looks as bellow:
> xcrun simctl launch --console-pty D955FAB6-C55D-41F9-9893-2A4121AD0991 com.sample.mobile.ios -rvm:log=trace

It produces tons of output but related to class initialization looks as bellow: 
```
Throwing a java/lang/NullPointerException. Call stack:
    org/robovm/apple/foundation/NSObject.alloc(Lorg/robovm/objc/ObjCClass;)J:185
    org/robovm/apple/foundation/NSObject.alloc()J:179
    org/robovm/objc/ObjCObject.<init>()V:86
    org/robovm/apple/foundation/NSObject.<init>(Lorg/robovm/apple/foundation/NSObject$SkipInit;)V:139
    org/robovm/apple/foundation/NSString.<init>(Ljava/lang/String;)V:108
    com/example/mobile/ios/Foo.<clinit>()V:27
    java/lang/Class.classForName(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;:-2
    java/lang/Class.forName(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;:218
    java/lang/Class.forName(Ljava/lang/String;)Ljava/lang/Class;:176
    org/robovm/objc/ObjCClass.<clinit>()V:78
    org/robovm/objc/ObjCObject.<clinit>()V:77
    com/example/mobile/ios/Main.testNSObject()V:9
    com/example/mobile/ios/Main.main([Ljava/lang/String;)V:14
```

NPE happens during `Foo` class static member initialization `npeHere = new NSString("NPE is here")` inside `NSObject`s code:
```
    private static final Selector alloc = Selector.register("alloc");
    static long alloc(ObjCClass c) {
        long h = ObjCRuntime.ptr_objc_msgSend(c.getHandle(), alloc.getHandle());
        if (h == 0L) {
            throw new OutOfMemoryError();
        }
        return h;
    }
``` 

At this line there are two options for NPU: either (or both) `c` to `alloc` is null. Long story short -- it was `alloc`. But from code it can't be normally `null`, but in reality can if `NSString` is not finished class initialization.

Back to trace log, to check sequence of class loading(simplified):
```json
{
    "Loading com/example/mobile/ios/Main": { },
    "Initializing com/example/mobile/ios/Main": {},
    "Loading org/robovm/apple/foundation/NSObject": {},
    "Initializing org/robovm/apple/foundation/NSObject": {
        "Initializing org/robovm/objc/ObjCObject": {
            "Loading org/robovm/objc/ObjCClass": {},
            "Initializing org/robovm/objc/ObjCClass": {
                "Loading com/example/mobile/ios/Foo": {},
                "Initializing com/example/mobile/ios/Foo": {
                    "Loading org/robovm/apple/foundation/NSString": {},
                    "Initializing org/robovm/apple/foundation/NSString": {},
                    "Exception java/lang/NullPointerException": [
                        "org/robovm/apple/foundation/NSObject.alloc(Lorg/robovm/objc/ObjCClass;)J:185",
                        "org/robovm/apple/foundation/NSObject.alloc()J:179",
                        "org/robovm/objc/ObjCObject.<init>()V:86",
                        "org/robovm/apple/foundation/NSObject.<init>(Lorg/robovm/apple/foundation/NSObject$SkipInit;)V:139",
                        "org/robovm/apple/foundation/NSString.<init>(Ljava/lang/String;)V:108",
                        "com/example/mobile/ios/Foo.<clinit>()V:27",
                        "java/lang/Class.classForName(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;:-2",
                        "java/lang/Class.forName(Ljava/lang/String;ZLjava/lang/ClassLoader;)Ljava/lang/Class;:218",
                        "java/lang/Class.forName(Ljava/lang/String;)Ljava/lang/Class;:176",
                        "org/robovm/objc/ObjCClass.<clinit>()V:78",
                        "org/robovm/objc/ObjCObject.<clinit>()V:77",
                        "com/example/mobile/ios/Main.testNSObject()V:9",
                        "com/example/mobile/ios/Main.main([Ljava/lang/String;)V:14"
                    ]
                }
            }
        }
    }
}
```

From call tree: `NSObject.alloc()` was called while `NSObject.<clinit>` even not started. As before `NSObject` it's super class has to be initialized first `ObjCObject` and before it -- `ObjCClass`. 
And at last `ObjCClass` does class pre-load right before in middle of this.

# Workaround 
It shall work if `ObjCClass` is loaded in calm condition, but sadly preload of it with `Class.forName("org.robovm.objc.ObjCClass")` fails with:  
```
java.lang.ExceptionInInitializerError
	at java.lang.Class.classForName(Native Method)
	at java.lang.Class.forName(Class.java:218)
	at java.lang.Class.forName(Class.java:176)
	at com.example.mobile.ios.Main.main(Main.java:8)
Caused by: java.lang.NullPointerException
	at org.robovm.objc.ObjCClass.getByType(ObjCClass.java:275)
	at org.robovm.objc.ObjCObject.<clinit>(ObjCObject.java:77)
	... 4 more
```

And here we have SAME cycle dependency: 
```
ObjCClass.<clinit>() {
   super.<clinit>() {  // ObjCObject
      ObjCClass.getByType()  // <<<< NPE inside it as ObjCClass is not initialized 
   } 
}   
```

As `ObjCClass` is final chances are low that it is going to be used something like this. As `ObjCClass.getByType()` inside `ObjCObject` is compiler generated it is better to leave things as they are.  
As `ObjCObject` has references to `ObjCClass` it might cause class being initialized, and this does work: 

Following code being added at very beginning of iOS code will fix the issue. 
> Class.forName("org.robovm.objc.ObjCObject"); // FIXME: workaround for Failed to preload inner ObjCClass xxxx

# The fix. Reverts MobiVM/robovm#754
### Reasons to revert:
- Crash itself;
- Another moment to revert: Jetbrains has fixed https://youtrack.jetbrains.com/issue/IDEA-332794 and delivered fix in 2025.2. (bad moment -- it fixes big Java case but there is still issue with Robo)
- These changes were applied only to NSObject subclasses but any inner Java classes affected.

# Fix
Revert was proposed as [#815](https://github.com/MobiVM/robovm/pull/815)
