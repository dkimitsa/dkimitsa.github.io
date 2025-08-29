---
layout: post
title: 'Swift: missing _OBJC_CLASS_$_RvmProduct_PurchaseOption case'
tags: [investigation, runtime, swift, binding]
---
[issue#813](https://github.com/MobiVM/robovm/issues/813) was reported as an issue with [storekit2]({{ site.baseurl }}{% post_url 2025-04-20-binding-storekit2-swift-framework %}) bindings.  

> Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '+[missing_RvmProduct_PurchaseOption appAccountToken:]: unrecognized selector sent to class 0x1332748a0'

With comments that simulator works but app fails on real device.  
Lets investigate.
<!-- more -->

## Background

First if all `RvmProduct_PurchaseOption` is ObjectiveC wrapper around pure swift class `Product.PurchaseOption` and it looks like bellow:  
```swift
@available(iOS 15.0, macOS 12.0, tvOS 15.0, watchOS 8.0, visionOS 1.0, *)
extension RvmProduct {
    @objc(RvmProduct_PurchaseOption)
    public class PurchaseOption : NSObject {
        let raw: Product.PurchaseOption
        init(raw: Product.PurchaseOption) {
            self.raw = raw
        }
        ....
    }
}
```

It has all required `@objc` annotations and should be available. 

### missing_RvmProduct_PurchaseOption
Exception report class with `missing_` prefix, means that `objc_getClass` was not able to resolve `RvmProduct_PurchaseOption` and stub was added instead(it's a [workaround]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %}) for missing intermediate classes]).
`objc_getClass` its ObjectiveC runtime API that returns pointer to class by its name. Class is available to runtime if:
- from `_OBJC_CLASS_$_ClassName` structure that compiler stores in binary;
- or class is created runtime using `objc_allocateClassPair` API. 

Class should be available in binary, checking both `iphone` and `simulator` versions:
```
nm -g StoreKitRvm.xcframework/ios-arm64/StoreKitRvm.framework/StoreKitRvm | grep 'OBJC_CLASS_$_RvmProduct_PurchaseOption'
nm -g StoreKitRvm.xcframework/ios-arm64_x86_64-simulator/StoreKitRvm.framework/StoreKitRvm | grep 'OBJC_CLASS_$_RvmProduct_PurchaseOption'
0000000000049ba0 S _OBJC_CLASS_$_RvmProduct_PurchaseOption
```
Symbol is missing in `iphone` version but is present in `simulator`'s that matches initial issue description.

### Tracking down the issue 
Long story short. It was isolated to following minimal conditions to reproduce the issue: 
- targeting `iphoneos`; 
- targeting ios12 `-target arm64-apple-ios12` (targeting ios13 doesn't have an issue);
- class contains property of type that belongs to higher iOS version than compilation target;

Minimal code to reproduce:
```swift
import Foundation
import StoreKit

@available(iOS 15.0, *)
@objc(RvmProduct_PurchaseOption) public class RvmProduct_PurchaseOption: NSObject  {
     let raw: Product.PurchaseOption
     init(raw: Product.PurchaseOption) {
        self.raw = raw 
     }
}
```

And command lines for demonstration: 
```
swiftc -c test.swift -sdk `xcrun --sdk iphoneos --show-sdk-path` -target arm64-apple-ios12 -o arm64-iphone-12.o 
swiftc -c test.swift -sdk `xcrun --sdk iphoneos --show-sdk-path` -target arm64-apple-ios13 -o arm64-iphone-13.o 
swiftc -c test.swift -sdk `xcrun --sdk iphonesimulator --show-sdk-path` -target arm64-apple-ios12.0-simulator -o arm64-sim.o
nm ./arm64-iphone-12.o | grep _OBJC_CLASS_  
nm ./arm64-iphone-13.o | grep _OBJC_CLASS_  
                 U _OBJC_CLASS_$_NSObject
0000000000001758 S _OBJC_CLASS_$_RvmProduct_PurchaseOption
nm ./arm64-sim.o | grep _OBJC_CLASS_  
                 U _OBJC_CLASS_$_NSObject
0000000000001728 S _OBJC_CLASS_$_RvmProduct_PurchaseOption
```

### Bottom line
It is looks like a bug of swift compiler. It is big question from other side why target ios12 when using ios15.  
As a fix it is enough to target more recent version.

Opened a topic on [forum.swift.org](https://forums.swift.org/t/objc-interop-objc-class-is-not-generated-when-using-target-arm64-apple-ios12-and-accessing-ios15-code/81873) to see if there will be any insights. 

### Happy coding  
