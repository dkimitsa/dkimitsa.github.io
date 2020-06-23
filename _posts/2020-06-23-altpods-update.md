---
layout: post
title: 'AltPods: pods updated - 1.8.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were update to v1.8.0-SNAPSHOTS to sync with recent releases. Part of list didn't receive any API update hovewer bindings were re-generated against recent version of frameworks. Update list look as bellow:

### New framework
- [Firebase/Authentication](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-auth) `v6.27.0`
- [Firebase/RemoteConfig](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-remoteconfig) `v6.27.0`

### Updated pods
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.12.8`
- [BranchMetrics](https://github.com/dkimitsa/robovm-robopods/tree/alt/branchmetrics/ios) `v0.34.0`
- [FacebookSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/facebook) `v7.0.1`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.27.0`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/alt/onesignal/ios) `v2.14.2`


These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.8.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
