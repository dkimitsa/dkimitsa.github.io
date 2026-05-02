---
layout: post
title: 'AltPods: pods updated - issue 1.59'
tags: ['bro-gen', binding, robopods, altpods]
---

Alt-pods update issue `1.59`, new sync with recent releases.  
Released to maven central. 

### Updated pods 
* Tenjin updated `1.15.1` to `1.16.1` 
* OneSignal updated `5.5.0` to `5.5.1`
* IronSource updated `9.2.0` to `9.4.1` 
* Fyber updated `8.4.5` to `8.4.7` 
* CleverAds updated `4.6.3` to `4.6.6`
* AppsFlyer updated `6.17.9` to `6.18.0` 
* Inmobi updated `11.1.1` to `11.2.0` 
* Google Mobile Ads updated `13.1.0` to `13.3.0` 
* Firebase updated to `v12.10.0` -> `v12.12.0` 
* new framework: Adjust SDK `v5.6.2`

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

