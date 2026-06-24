---
layout: post
title: 'AltPods: pods updated - issue 1.60'
tags: ['bro-gen', binding, robopods, altpods]
---

Alt-pods update issue `1.60`, new sync with recent releases.  
Released to maven central. 

### Updated pods 
* Google Mobile Ads adapters updated 
* Google Mobile Ads updated from `13.3.0` to `13.5.0`
* UnityAds updated from `4.17.0` to `4.18.1` (no api changes) 
* Tenjin updated from `1.16.1` to `1.17.1` 
* Singular updated from `12.10.1` to `12.12.0` 
* OneSignal updated from `5.5.1` to `5.5.3` 
* Lottie updated from `4.6.0` to `4.6.1` 
* InMobiSDK updated from `11.2.0` to `11.3.0` 
* Firebase updated from `12.2.0` to `12.15.0` 
* CleverAds updated from `4.6.6` to `4.7.4`
* Chars updated, no api/version changes 
* AppsFlyer updated from `6.18.0` to `7.0.0` 
* AppLovinSDK updated from `13.6.2` to `13.6.3` (no api changes) 
* AdjustSDK updated from `5.6.2` to `5.7.0` 

### about versioning 
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

* These pods were released to Maven central, each pod under own version.
  [Source code @github](https://github.com/dkimitsa/robovm-robopods/)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

