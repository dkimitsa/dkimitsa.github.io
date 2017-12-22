---
layout: post
title: 'bro-gen: binding Swift framework, but not so fast!'
tags: ['bro-gen', binding, fix, swift]
---

It was not big deal to for me binding framework with bro-gen ([check tutorial)]({{ site.baseurl }}{% post_url 2017-10-19-bro-gen-tutorial %}) until I faced one Swift lib - [Charts](https://github.com/danielgindi/Charts) to be exact. It was problematic and exposed few robovm/bro-gen problems:

<!-- more -->
- `objective-c` header has bunch of classes with duplicate property names e.x. `normalizeSizeEnabled` and `isNormalizeSizeEnabled` at same time. bro-gen will add `is` prefix to `normalizeSizeEnabled` as it is boolean and turn it into `isNormalizeSizeEnabled`, which causes conflict with already existing `isNormalizeSizeEnabled`. That is a pain as there are dozen such places and it resulted into many manual edits in yaml file;
- `bro-gen` is not able to understand property of block type, e.x.  
`@property (nonatomic, copy) CGFloat (^ _Nullable block)(id <ILineChartDataSet> _Nonnull, id <LineChartDataProvider> _Nonnull);`.  
There is no solution for this problem today as it is in underlying `ffi-clang`. So this property was just manually excluded;
- issue with RoboVM and swift integration. Once `RoboVM` detects that framework uses `Swift` it copies all swiftLibs it depends on. But Swift library might have own dependencies on other swiftLibs and these dependencies were not detected and not copied into application bundle which caused crash at runtime. [Issue #244](https://github.com/MobiVM/robovm/issues/244) and [fix PR#245](https://github.com/MobiVM/robovm/pull/245). Crash log is similar to bellow:
```
dyld: Library not loaded: @rpath/libswiftos.dylib
  Referenced from: /Users/dkimitsa/Library/Developer/CoreSimulator/Devices/71125D31-29BD-4C10-AADC-A64529080DA8/data/Containers/Bundle/Application/87D23FB4-8D98-4AB3-98D0-C67E0CF2BEE5/Main.app/Frameworks/libswiftMetal.dylib
  Reason: image not found
```
- once started it was discovered that `Swift` runtime class name could be different than declared in `objc .h` files which causes crash on runtime. For example:
```objc
/// View that represents a pie chart. Draws cake like slices.
SWIFT_CLASS("_TtC6Charts12PieChartView")
@interface PieChartView : PieRadarChartViewBase
```
At RoboVM side this has to be explicit specified in @NativeClass annotation:
```java
/*</javadoc>*/
/*<annotations>*/@Library(Library.INTERNAL) @NativeClass("_TtC6Charts12PieChartView")/*</annotations>*/
/*<visibility>*/public/*</visibility>*/ class /*<name>*/PieChartView/*</name>*/
    extends /*<extends>*/PieRadarChartViewBase/*</extends>*/
    /*<implements>*//*</implements>*/ {
```
The problem was here that bro-gen was not able to produce such annotations and it was extended with it, [check commit](https://github.com/dkimitsa/robovm-bro-gen/commit/2999a373bbc1006ffe79072497e8700c0e5d9a71)
- on the way following moment was fixed: old known  
`[WARNING] Failed to find Java classes for the following Objective-C classes in Storyboard/XIB files: [UIResponder, PieChartView]`  
It was not interesting for me for long time as long as it was complain for `UIResponder` mostly. But not it complained for class that was present and warning was wrong as it was not causing any crash. Long story short: RoboVM was considered only @CustomClass annotated java classes as candidates for XIB's customs classes. But it should also look into @NativeClass as well. In this example `PieChartView` is native class from third party framework. [issue #246](https://github.com/MobiVM/robovm/issues/246) and [fix pr#247](https://github.com/MobiVM/robovm/issues/247)

`Charts` bindings will be released soon.
