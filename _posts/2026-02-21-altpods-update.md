layout: post
title: 'AltPods: pods updated - issue 1.57'
tags: ['bro-gen', binding, robopods, altpods]
---

Alt-pods update issue `1.57`, new sync with recent releases and first update after migrating to new version numbering. 
### how version builds 
Each pod has version of corresponding framework + `.0` suffix. E.g.:
`Google Mobile Ads 13.0.0` are accessible as following dependency:
> implementation "io.github.dkimitsa.robovm:robopods-google-mobile-ads-ios:13.0.0.0"

Also for set of pods, built from single source there are bill-of-material available, where version of bom is framework version with `.0` suffix:  
*Facebook*: 
> implementation platform("io.github.dkimitsa.robovm:robopods-facebook-bom:18.0.3.0) // v18.0.3  
> implementation "io.github.dkimitsa.robovm:robopods-facebook-login-ios"

*Firebase*:
> implementation platform("io.github.dkimitsa.robovm:robopods-firebase-bom:12.9.0.0") // v12.9.0  
> implementation "io.github.dkimitsa.robovm:robopods-firebase-analytics-ios"


### Updated pods (ones with API changed)
* Singular updated `12.9.0` to `12.10.0`
* OneSignal updated `5.2.15` to `5.4.1`
* Fyber updated `8.4.2` to `8.4.5`
* CleverAds updated `4.5.4` to `4.6.2`
* BranchMetrics updated `3.13.0` to `3.14.0` 
* Inmobi updated `11.1.0` to `11.1.1`
* ApplovinSDK updated `13.5.1` to `13.6.0`
* Google Mobile Ads updated `12.14.0` to `13.0.0`
* Facebook updated `18.0.1` to `18.0.3`
* Firebase updated to `v12.7.0` -> `v12.9.0`

### Updated pods (ones with API changed)
* UnityAds updated `4.16.5` to `4.16.6`
* Tenjin updated `1.15.0` to `1.15.1`
* Lottie updated `4.5.2` to `4.6.0`

* These pods were were released to Maven central, each pod under own version.
  [Source code @github](https://github.com/dkimitsa/robovm-robopods/)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

