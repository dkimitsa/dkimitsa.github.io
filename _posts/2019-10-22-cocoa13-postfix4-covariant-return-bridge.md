---
layout: post
title: "iOS13 PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method"
tags: ["fix", "compiler", "postfix"]
---
This post continues the series of fixes discovered during compilation of CocoaTouch library.  
# PostFix #4: @Bridge annotated covatiant return bridget methods

Other postfixes:  
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})

Issue was discovered while compiling `NWTxtRecord`: 
```
java.lang.IllegalArgumentException: @Bridge annotated method <org.robovm.apple.network.NWTxtRecord: org.robovm.apple.foundation.NSObject copy()> must be native
	at org.robovm.compiler.BridgeMethodCompiler.validateBridgeMethod(BridgeMethodCompiler.java:87)
	at org.robovm.compiler.BridgeMethodCompiler.doCompile(BridgeMethodCompiler.java:233)
```

## Root case and fix
In source code `NWTxtRecord` doesn't contain `NSObject copy()` method but containts `NWTxtRecord copy()`.  
Hovewer once class file is decompiled following methods can be found there:  
<!-- more -->
```
  public native org.robovm.apple.network.NWTxtRecord copy();
    descriptor: ()Lorg/robovm/apple/network/NWTxtRecord;
    flags: (0x0101) ACC_PUBLIC, ACC_NATIVE
    RuntimeVisibleAnnotations: org.robovm.rt.bro.annotation.Bridge(symbol="nw_txt_record_copy")

  public org.robovm.apple.foundation.NSObject copy();
    descriptor: ()Lorg/robovm/apple/foundation/NSObject;
    flags: (0x1041) ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    RuntimeVisibleAnnotations: org.robovm.rt.bro.annotation.Bridge(symbol="nw_txt_record_copy")
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokevirtual #13                 // Method copy:()Lorg/robovm/apple/network/NWTxtRecord;
         4: areturn
```

`NSObject copy()` is present in class file and was added there by `javac` to handle overriden method with covariant return type. As `NWTxtRecord` extends `NSObject` at some level and `NSObject` contains `copy()` method with return type that covariant to `NWTxtRecord`. More details in [Oracle's document](https://www.oracle.com/technetwork/java/jvmls2013heid-2013922.pdf).

RoboVM compiler finds `@Bridge` annotation on this synthetic method (as these being copied since [JDK-6695379](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=6695379)) and tries to handle it as a @Bridge to native founction and it fails as method doesn't match the criteria (currently it is not native).

There are two way of fix: 
- simple, as @Bridge methods are just binding for native function the name of method might be changed to eliminate override with covariant return type case;
- modify the compiler to handle these methods, skips them actually. 


Second approach was implemented and delivered as [PR422](https://github.com/MobiVM/robovm/pull/421)

