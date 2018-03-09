---
layout: post
title: "W/L: Adding swift libs to iOS sdk"
tags: ["linux windows", swift]
---
RoboVM needs swift libraries as these has to be embedded into application in case any dependency is using swift. E.g. in case you favorite framework is built using swift. Precompiled libraries are available from https://swift.org. Seems to be easy but not so fast. Details bellow.  
<!-- more -->

Once libraries from `swift.org` are put in their place in SDK any run will fails with following:
```
dyld: Library not loaded: @rpath/libswiftCore.dylib
  Referenced from: /private/var/containers/Bundle/Application/C52BDDFF-CF23-4411-B4FC-50129F292EE7/Main.app/Frameworks/Charts.framework/Charts
  Reason: Incompatible library version: Charts requires version 1.0.0 or later, but libswiftCore.dylib provides version 0.0.0
```

Lets check dependency requirement of `Charts.framework/Charts`:
```
$ otool -L Charts
Charts:
	@rpath/Charts.framework/Charts (compatibility version 1.0.0, current version 1.0.0)
    ...
	@rpath/libswiftCore.dylib (compatibility version 1.0.0, current version 900.0.74)
    ...
```

And lets check the version of `libswiftCore.dylib` from `swift.org`:  
```
$ otool -l ~/.robovm/platform/Xcode.app/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib/swift/iphoneos/libswiftCore.dylib | grep -A 5 LC_ID_DYLIB
          cmd LC_ID_DYLIB
      cmdsize 52
         name @rpath/libswiftCore.dylib (offset 24)
   time stamp 1 Thu Jan  1 03:00:01 1970
      current version 0.0.0
compatibility version 0.0.0
```

Here zeroes comes. The reason for this found at [apple forum](https://forums.developer.apple.com/thread/64453):
>Currently, Swift does not support building frameworks that are distributed as binaries. All Swift code in all targets of an app must be compiled with the same version of Swift.
 This is because the Swift ABI (application binary interface) is not locked down yet, so the rules change with every version of the Swift compiler. In addition the Swift library itself is still changing, so even if the ABI was compatible, the library functions would be different.

## Making it working with RoboVM
Explanation is interesting but it doesn't help with RoboVM. To enable libraries to be working with RoboVM I write [simple java tool](https://github.com/dkimitsa/robovm-toolchain/blob/master/xcode/tools/ForceVersion.java) then sets version information by updating LC_ID_DYLIB command. After setting version=4.0.1 and compatibility=1.0.0 swift frameworks started available for builds from Linux/Windows.

## Source code
All changes are available as part of [dkimitsa/robovm-toolchain](https://github.com/dkimitsa/robovm-toolchain) repository and in particular of [this commit](https://github.com/dkimitsa/robovm-toolchain/commit/1568a691407fbdc0439cd553e43266e3128619cf).
