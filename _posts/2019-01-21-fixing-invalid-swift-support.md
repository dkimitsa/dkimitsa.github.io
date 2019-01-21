---
layout: post
title: 'Fix: Xcode10.1 - Invalid Swift Support'
tags: [fix, swift]
---
Starting from Xcode 10 [ARM64E arch has to be stripped from included framworks](https://stackoverflow.com/questions/53214530/invalid-swift-support-files-dont-match-xcode-10-1) as this caused `Invalid Swift Support` at Apple side. There was a [fix for it](https://github.com/MobiVM/robovm/pull/340) but sadly there was no feedback and another was introduced during the fix. As result investigation continue with [Eric Nondahl report](https://gitter.im/MobiVM/robovm?at=5c3e25801cb70a372ae35d9a) and fix is ongoing (not confirmed yet).  

To fix this issue [PR346](https://github.com/MobiVM/robovm/pull/346) is created and it introduces following changes:  
* fixes issue introduced in [PR340](https://github.com/MobiVM/robovm/pull/340): copy of swift libraries in `SwiftSupport/` was stripped from ARM64E arch but shell not be touched at all;
* swift libraries are now being copied into `SwiftSupport/iphoneos/` same as XCode does;
* added code that strips all not used architectures from dynamic binaries (check [rules_apple](https://github.com/bazelbuild/rules_apple/commit/b238ee53afd6c70eed5a9346dbd834f09f96b89f) for reference). Probably it is extra as it is enough to remove simulator archs and ARM64E but it is quick win as reduces IPA size. So if it is not breaking anything lets keeping it;
* added code that strips bitcode from dynamic binaries. Same as Xcode does. Reduces final IPA a lot; 
* as now all extra architectures are being stripped there is no need for `stripArchs` configuration option that was introduced in [PR340](https://github.com/MobiVM/robovm/pull/340). It was removed. 

Solution is in progress, please report to [gitter channel](https://gitter.im/MobiVM/robovm) for any issue found.