---
layout: post
title: "iOS13 PostFix #8: workaround for missing objc classes(ObjCClassNotFoundException)"
tags: ["binding", "postfix"]
---
Apple altogether with introducing new APIs also keeps refactoring existing one. In this case it results in part of API being extracted from class and moved to newly introduced class. And this class is being used as new super.
Result class will have schema when super class introduced later than child. This makes problem when running code on iOS older than super was introduced.
Bright example:
```objc
// since ios 4.1
@interface GKPlayer : GKBasePlayer

// but

// since ios 10.0
@interface GKBasePlayer : NSObject
```

## Root case
<!-- more -->
RoboVM bro bridge resolves native classes for binded java ones and if class is missing it throws `ObjCClassNotFoundException`.
Existing bindings corrected this behaviour by manually specifying `NSObject` as super. Also it might require manual copying methods from unsupported super. With iOS13 there are lot of such classes with supers younger than children. Manual corrections becomes problematic and also this leads to lost of common parent class.

## The fix (workaround)
How it expected to work:
### it should allow to have possibly missing objc classes as parent;
For each missing obj-c class case instead of throwing `ObjCClassNotFoundException` new objc class will be registered. It will inherit expected super of missing class. Java class will use this as its shadowed objc class.

### it should not allow direct instantiation of missing classes.
New class for missing one will override `+alloc` method and will throw `ObjCClassNotFoundException`. This makes impossible direct instantiation of missing classes.

It will work as objc runtime doesn't bother about java side and all native objc methods invocations results in `objc_msgsend`.
Code is delivered as [PR442](https://github.com/MobiVM/robovm/pull/442)

## Other postfixes:
<!-- more -->
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})
* [PostFix #9: experimental and formal bitcode support]({{ site.baseurl }}{% post_url 2019-12-26-cocoa13-postfix9-formal-bitcode-support %})
