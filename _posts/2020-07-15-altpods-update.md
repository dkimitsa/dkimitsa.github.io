---
layout: post
title: 'AltPods: pods updated - 1.9.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were updated to v1.9.0-SNAPSHOTS to sync with recent releases. Part of list didn't receive any API update hovewer bindings were re-generated against recent version of frameworks. Update list look as bellow:

### New frameworks
- [Flurry/Analytics](https://github.com/dkimitsa/robovm-robopods/tree/alt/flurry/ios-analytics) `v10.3.3`
- [Flurry/Ads](https://github.com/dkimitsa/robovm-robopods/tree/alt/flurry/ios-ads) `v10.3.3`
- [Flurry/Config](https://github.com/dkimitsa/robovm-robopods/tree/alt/flurry/ios-config) `v10.3.3`
- [Flurry/Messaging](https://github.com/dkimitsa/robovm-robopods/tree/alt/flurry/ios-messaging) `v10.3.3`
- [Firebase/Messaging](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-messaging) `v4.6.0 (firebase v6.27.0)`

### Updated pods
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.13.1`
- [FacebookSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/facebook) `v7.1.1`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.28.0`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/alt/onesignal/ios) `v2.14.3`


These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.9.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
