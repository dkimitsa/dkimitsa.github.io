---
layout: post
title: 'RoboVM 2.3.16 release and 2.3.17-SNAPSHOT'
tags: [whatsnew]
---
Idea plugin is not available at [Idea Marketplace](https://plugins.jetbrains.com/plugin/14440-mobivm) yet as in review!  
But can be downloaded manually from [MobiVM](https://mobivm.github.io/dev/) site.

## 2.3.16: What's new
* Fixed: 'duplicate symbol xxx.spfpoffset' in debug build when using GoogleMobileAds pod [PR626](https://github.com/MobiVM/robovm/pull/626)
* Framework target: can produce XCFramework, can produce m1 simulator slice [PR624](https://github.com/MobiVM/robovm/pull/623)
* Fixed: missing bitcode in VM libs (introduced by m1 support changes) [PR624](https://github.com/MobiVM/robovm/pull/624)
* Fixed [#621](https://github.com/MobiVM/robovm/issues/621) -- hang of SKStoreReviewController.requestReview  by [PR615](https://github.com/MobiVM/robovm/pull/622). **WARNING**: window is to be retained in user code now !!!
* ByteBuffer J9 API desugaring [PR615](https://github.com/MobiVM/robovm/pull/615)
* ios15 binding [PR613](https://github.com/MobiVM/robovm/pull/613)

## 2.3.17-SNAPSHOT:
* Fixed: missing simulator arch in Idea picker [PR642](https://github.com/MobiVM/robovm/pull/642)
* New: ios15.4 bindings [PR635](https://github.com/MobiVM/robovm/pull/635)
* Changes: to `swiftSupport` configuration parameter [PR638](https://github.com/MobiVM/robovm/pull/638)
* Changes: Debugger -- can suspend any thread [PR628](https://github.com/MobiVM/robovm/pull/628)

Happy coding!  
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).
