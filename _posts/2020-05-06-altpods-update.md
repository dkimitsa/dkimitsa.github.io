---
layout: post
title: 'AltPods: pods updated - 1.7.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were update to v1.7.0-SNAPSHOTS to sync with recent releases. Part of list didn't receive any API update hovewer bindings were re-generated against recent version of frameworks. Update list look as bellow:

### New framework
- [Firebase/Crashlytics](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase/ios-crashlytics) `v6.24.0`

### Updated pods
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.12.4`
- [BranchMetrics](https://github.com/dkimitsa/robovm-robopods/tree/alt/branchmetrics/ios) `v0.33.1`
- [Charts](https://github.com/dkimitsa/robovm-robopods/tree/alt/charts/ios) `v3.5.0`
- [FacebookSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/facebook) `v7.0.0`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.24.0`
- [HelpShift](https://github.com/dkimitsa/robovm-robopods/tree/alt/helpshift/ios) `v7.7.1`
- [Lottie](https://github.com/dkimitsa/robovm-robopods/tree/alt/lottie/ios) `v3.1.8`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/alt/onesignal/ios) `v2.31.1`

### Unchanged pods
- [Pollfish](https://github.com/dkimitsa/robovm-robopods/tree/alt/pollfish/ios) `v5.2.5`
- [SAMKeychain](https://github.com/dkimitsa/robovm-robopods/tree/alt/samkeychain/ios) `v1.5.3`


These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.7.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods)


Updates are not fullty tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
