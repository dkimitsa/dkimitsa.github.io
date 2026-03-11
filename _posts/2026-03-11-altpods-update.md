---
layout: post
title: 'AltPods: pods updated - issue 1.58'
tags: ['bro-gen', binding, robopods, altpods]
---

Alt-pods update issue `1.58`, new sync with recent releases and first update after migrating to new version numbering. 

### Updated pods (ones with API changed)
* Appsflyer updated `6.17.8` to `6.17.9`
* CleverAds updated `4.5.4` to `4.6.2`
* Firebase updated to `v12.9.0` -> `v12.10.0`
* Google Mobile Ads updated `13.0.0` to `13.1.0`
* OneSignal updated `5.4.1` to `5.5.0`
* UnityAds updated: `v4.16.6` -> `v4.17.0`

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

