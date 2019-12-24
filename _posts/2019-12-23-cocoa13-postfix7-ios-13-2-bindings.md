---
layout: post
title: "iOS13 PostFix #7: bindings for ios13.2"
tags: ["binding", "bro-gen", "postfix"]
---
This post continues the series of fixes discovered during compilation of CocoaTouch library and improvements to compiler.
# PostFix #7: iOS 13.2 bindings

While things being fixes iOS 13.2 is already in wild. This postfix delivers:
- changes to support new api;
- fixes in javadoc;
- fixes to structures that have to be annotated as packed.

Code is delivered as [PR441](https://github.com/MobiVM/robovm/pull/441)

## Other postfixes:
<!-- more -->
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #8: workaround for missing objc classes(ObjCClassNotFoundException)]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %})
