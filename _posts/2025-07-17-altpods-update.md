---
layout: post
title: 'AltPods: pods updated - 1.51.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were updated to v1.51.0-SNAPSHOT to sync with recent releases.

### Breaking changes
AdMob is not longer a part of Firebase. It was moved out of Firebase package so its artifact id is changed:
- admob -> `io.github.dkimitsa.robovm:robopods-google-mobile-ads-ios`
- ump -> `io.github.dkimitsa.robovm:robopods-google-ump-ios`
- DynamicLinks removed as not available in Firebase anymore.

Silly typo fixed in `CleverAds` class package name: `package org.robovm.pods.cleverads.*` now.

### Updated pods (ones with API changed)
- [AppsFlyer](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.51.0/appsflyer)   `v6.17.0` to `v6.17.2`
- [CleverAds](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.51.0/cleverads/)  `v4.1.0` to `v4.1.2`
- [Google Mobile Ads](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.51.0/google-mobile-ads/ios-google-mobile-ads) `v12.5.0` to `v12.7.0`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.51.0/firebase/)       `v11.14.0` -> `v12.0.0`

### Updates without any API change
* Fyber updated `v8.3.7` to `v8.3.8` 
* UnityAds updated: `v4.15.0` to `v4.15.1` 

* These pods were pushed to `https://central.sonatype.com/repository/maven-snapshots/` maven repo under `1.51.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.51.0)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.
