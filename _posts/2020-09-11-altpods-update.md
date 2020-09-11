---
layout: post
title: 'AltPods: pods updated - 1.11.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were updated to v1.11.0-SNAPSHOTS to sync with recent releases. Part of list didn't receive any API update hovewer bindings were re-generated against recent version of frameworks. Update list look as bellow:

### New pods
- [Google Ads Mediation adapters](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-google-mobile-ads-adapters) 
- [Google User Messaging Platform](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-google-ump) 
- [InMobi](https://github.com/dkimitsa/robovm-robopods/tree/alt/inmobi/ios-sdk) `9.0.7`
- [Fyber Marketplace](https://github.com/dkimitsa/robovm-robopods/tree/alt/inmobi/ios-sdk) `7.6.4`

### Updated pods
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.14.2`
- [BranchMetrics](https://github.com/dkimitsa/robovm-robopods/tree/alt/branchmetrics/ios) `0.35.0`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.32.0`
- [Flurry](https://github.com/dkimitsa/robovm-robopods/tree/alt/flurry) `v11.0.0`


These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.11.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
