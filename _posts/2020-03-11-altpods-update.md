---
layout: post
title: 'AltPods: pods updated - 1.6.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were update to v1.6.0-SNAPSHOTS to sync with recent releases. Part of list didn't receive any API update hovewer bindings were re-generated against recent version of frameworks. Update list look as bellow:

### New framework
- [Firebase/Google Mobile Ads](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-google-mobile-ads) `v6.18.0`

### Updated pods
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.11.5`
- [FacebookSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/facebook) `v6.2.0`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.18.0`
- [HelpShift](https://github.com/dkimitsa/robovm-robopods/tree/alt/helpshift/ios) `v7.7.0`


### Unchanged pods
- [BranchMetrics](https://github.com/dkimitsa/robovm-robopods/tree/alt/branchmetrics/ios) `v0.34.0`
- [Charts](https://github.com/dkimitsa/robovm-robopods/tree/alt/charts/ios) `v3.4.0`
- [Lottie](https://github.com/dkimitsa/robovm-robopods/tree/alt/lottie/ios) `v3.1.6`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/alt/onesignal/ios) `v2.11.6`
- [Pollfish](https://github.com/dkimitsa/robovm-robopods/tree/alt/pollfish/ios) `v5.2.5`
- [SAMKeychain](https://github.com/dkimitsa/robovm-robopods/tree/alt/samkeychain/ios) `v1.5.3`


These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.6.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods)


Updates are not fullty tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
