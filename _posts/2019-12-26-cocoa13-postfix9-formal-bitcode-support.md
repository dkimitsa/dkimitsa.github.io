---
layout: post
title: "iOS13 PostFix #9: experimental and formal bitcode support"
tags: ["bitcode", "compiler", "postfix"]
---
Bitcode was considered as big question once Apple introduced support for it. Mostly it was seen as danger for RoboVM project as Apple might use it as a blocking requirement and not allow application to be submitted without bitcode. While today lot of people think bitcode is not deliver significant benefits (if apple re-compiles apps on own servers) having it might be good idea due:  
- developers need it due fear;
- it is required when producing Framework target with RoboVM and including framework in bitcode-enabled project;
- it was a part of commercial RoboVm 1.14 compier.

## How support works
There is nice post [Embedded Bitcode](https://jonasdevlieghere.com/embedded-bitcode/) by Jonas Delieghere that covers basics. In case of RoboVM getting bitcode is not big deal as its an optimized form of LLVM IR that RoboVM compiler produces while building native code. The question is how to get it emited to object file. From post there are two option to mark object file as having bitcode: 
* Full support: a section `__LLVM, __bitcode` for storing the (optimized) bitcode. This is plain, binary bitcode, not wrapped in an archive as there's only one file for each object;
* Formal support: an empty section `__LLVM, __asm` to differentiate objects without bitcode from those built from assembly.

This post fix implemented formal support due following moment: 
* to add bitcode as section to obj file would require implementing functionality similar to clang's `-fembed-bitcode` as LLVM either produces bitcode, object or assemble file;
* RoboVM compiler does produce object file from assember file (it produces it from LLVM IR to hack method size fields in info object);
* genereting bitcode and allowing Apple to re-compile app will break line number and method information. As it being received from DWARF debug one, once recompiled by apple -- offsets might not match anymore. And it would require to disable line number information in stack traces. 

## What was done
* native libraries were re-compiled with `-fembed-bitcode` flags;
* each object file generated from Java class now gets `__LLVM, __asm` section;
* linker is being instructed to use `-fembed-bitcode`;
* as runtime libraries are huge now due included bitcode -- it being stripped of if not enabled to minimize debug footprint;

## How to use 
Bitcode support can be enabled: 
* in `robovm.xml` with `<enableBitcode>true</enableBitcode>`;
* in `robovm` setion of `build.gradle` with `enableBitcode = true`;
* in `enableBitcode` parameter to maven plugin;

Code is delivered as [PR443](https://github.com/MobiVM/robovm/pull/443)

## Other postfixes:
<!-- more -->
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})
* [PostFix #8: workaround for missing objc classes(ObjCClassNotFoundException)]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %})
