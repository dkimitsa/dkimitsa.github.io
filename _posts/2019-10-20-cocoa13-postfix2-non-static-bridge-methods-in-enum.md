---
layout: post
title: "iOS13 PostFix #2: Non static @Bridge method in enums"
tags: ["fix", "compiler", "postfix", "bro-gen"]
---
This post continues series of fixes discovered during compilation of CocoaTouch library.
# PostFix #2: Adding support for non-static @Bridge method in enums classes

Other postfixes:
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})
* [PostFix #8: workaround for missing objc classes(ObjCClassNotFoundException)]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %})
* [PostFix #9: experimental and formal bitcode support]({{ site.baseurl }}{% post_url 2019-12-26-cocoa13-postfix9-formal-bitcode-support %})
* [PostFix #10: glkit -- missing functions (static inline now)]({{ site.baseurl }}{% post_url 2020-01-03-cocoa13-postfix10-glkit-adding-missing-inline-fn %})

Issue was discovered during compiling of `UIImageSymbolWeight` class:
```
java.lang.IllegalArgumentException: Receiver of non static @Bridge method <org.robovm.apple.uikit.UIImageSymbolWeight: double toFontWeight()> must either be a Struct or a NativeObject
	at org.robovm.compiler.BroMethodCompiler.getBridgeOrCallbackFunctionType(BroMethodCompiler.java:554)
	at org.robovm.compiler.BroMethodCompiler.getBridgeFunctionType(BroMethodCompiler.java:526)
```

##  What is wrong
<!-- more -->
### Small background
The cause method is described as following:
```java
    /**
     * @since Available in iOS 13.0 and later.
     */
    @Bridge(symbol="UIFontWeightForImageSymbolWeight", optional=true)
    public native @MachineSizedFloat double toFontWeight();
```

It is not a part of enum but `UIFontWeightForImageSymbolWeight` function bindings was put into this valued enum by bro-gen, yaml file of binding looks as below:
```yaml
founctions:
    #ios13
    UIFontWeightForImageSymbolWeight:
        class: UIImageSymbolWeight
        name: toFontWeight
```

And the function itself is declared in UIKit as below:
```objc
UIKIT_EXTERN UIFontWeight UIFontWeightForImageSymbolWeight(UIImageSymbolWeight symbolWeight) API_AVAILABLE(ios(13.0),tvos(13.0),watchos(6.0));
```

### Where non static @Bridge method come from
Normally binding for C function goes as static method and binding for `UIFontWeightForImageSymbolWeight` should be:
```java
    @Bridge(symbol="UIFontWeightForImageSymbolWeight", optional=true)
    public static native @MachineSizedFloat double toFontWeight(UIImageSymbolWeight symbolWeight);
```

But in case receiver (class we put binding of function) and first argument's type matches method might be made non-static and without first parameter. RoboVM compiler handles the case and provides this as first argument when makes a call to C function.

## Root case
Is stated in execption text itself: *Receiver of non static @Bridge method must either be a Struct or a NativeObject.*
From other side when method is static and first parameter present compiler doesn't compline much as long as it can marshal the type. This is limitation a limitation of code that handles non-static @Bridge methods.

## The fix
Utlimate fix would be to adapt compiler to accept any marshalable type as receiver but this would require sensitive amount of code to be adapted. Real world fix is to support ValuedEnum type in row of Struct and NativeObject and provide proper marshallers. Its minimze amount of compiler code to be changed and enough to have it working.
Code delivered as [PR420](https://github.com/MobiVM/robovm/pull/420)
