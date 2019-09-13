---
layout: post
title: 'AltPods: pods updated + new bindings(Facebook)'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were update to v1.3.0-SNAPSHOTS. Changes include:

### New pod
- [Facebook: core, login, share, place, audience](https://github.com/dkimitsa/robovm-robopods/tree/alt/facebook) `v5.5.0` 

### Updated pod
- [AppLovinSDK](https://github.com/dkimitsa/robovm-robopods/tree/alt/applovinsdk/ios) `v6.9.1` 
- [BranchMetrics](https://github.com/dkimitsa/robovm-robopods/tree/alt/branchmetrics/ios) `v0.28.1`
- [Firebase](https://github.com/dkimitsa/robovm-robopods/tree/alt/firebase) `v6.8.1`
- [HelpShift](https://github.com/dkimitsa/robovm-robopods/tree/alt/helpshift/ios) `v7.6.2`
- [Lottie](https://github.com/dkimitsa/robovm-robopods/tree/alt/lottie/ios) `v3.1.2`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/alt/onesignal/ios) `v2.11.0`

### Unchanged pod
- [SAMKeychain](https://github.com/dkimitsa/robovm-robopods/tree/alt/samkeychain/ios) `v1.5.3`
- [Pollfish](https://github.com/dkimitsa/robovm-robopods/tree/alt/pollfish/ios) `v5.0.0`
- [Charts](https://github.com/dkimitsa/robovm-robopods/tree/alt/charts/ios) `v3.3.1`

### Other changes
All pods now have proper dependency setup:  
- references to other pods;
- references for required framework thorugh META-INF/robovm.xml (no need to specify frameworks in host application)

These pods were pushed to `https://oss.sonatype.org/content/repositories/snapshots` maven repo under `1.3.0-SNAPSHOTS` version.  
[Source code @github](https://github.com/dkimitsa/robovm-robopods)  



Updates are not fullty tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.

NB: AltPods -- are robopods kept at my personal standalone repo. This is done to prevent main robopods repo from turning into code graveyard. As these pods have low interest of community and low chances to be updated.
