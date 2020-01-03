---
layout: post
title: "iOS13 PostFix #10: glkit -- missing functions (static inline now)"
tags: ["binding",  "postfix"]
---
Report in [gitter](https://gitter.im/MobiVM/robovm?at=5e08d0d4b4ed68096efa54ac) shown another problem in bindings:
``` java.lang.UnsatisfiedLinkError: Optional @Bridge method GLKMatrix4.create(FloatBuffer;)GLKMatrix4; not bound ```

Quick investigation shown that this method is not available runtime anymore as tons of functions converted into static inline. Today this functions looks as bellow:
```
GLK_INLINE GLKMatrix4 GLKMatrix4MakeWithArray(float values[16])
{
    GLKMatrix4 m = { values[0], values[1], values[2], values[3],
                     values[4], values[5], values[6], values[7],
                     values[8], values[9], values[10], values[11],
                     values[12], values[13], values[14], values[15] };
    return m;
}
```

As result RoboVM is not able to resolve it runtime and use it.
As long as it is not available runtime there are two options to use it:
- get all API and .c file, build it into object file and distribute as part of bindings;
- implement missing API in java (porting it).

While first option is preferable due performance concerns second approach will be implemented as quickest solutions. All changes were added to [ios13.2 bindings branch](https://github.com/MobiVM/robovm/pull/441) as [commit](https://github.com/MobiVM/robovm/pull/441/commits/5219b09545462d0a56c8a38f26cc4710707375b4).

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
* [PostFix #9: experimental and formal bitcode support]({{ site.baseurl }}{% post_url 2019-12-26-cocoa13-postfix9-formal-bitcode-support %})
