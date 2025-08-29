---
layout: post
title: 'AltPods: pods updated - 1.50.0-SNAPSHOT'
tags: ['bro-gen', binding, robopods, altpods]
---
AltPods were updated to v1.50.0-SNAPSHOT to sync with recent releases.

*IMPORTANT*: Legacy OSSRH maven repository was migrated to Central Portal, make sure to update snapshot repository from:  
> maven { url 'https://oss.sonatype.org/content/repositories/snapshots' }  

to 
> maven { url 'https://central.sonatype.com/repository/maven-snapshots' }

### Updated pods (ones with API changed)
- [CleverAds](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/cleverads/)  `v4.0.2.1` to `v4.1.0`
- [Facebook Ads](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/facebook/ios-audience)   `v6.17.1` to `v6.20.0`
- [IronSource](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/ironsource/)   `v8.8.0` to `v8.9.1`
- [Lottie](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/lottie/)       `v4.5.1` to `v4.5.2`
- [OneSignal](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/onesignal/)         `v5.2.10` to `v5.2.14`
- [Singular](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/singular/)   `v12.8.0` to `v12.8.1`
- [UnityAds](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0/unityads/)           `v4.14.2` to `v4.15.0`


### Pods without any API changes due update
- AppLovinSDK updated: `v13.3.0` -> `v13.3.1` (no api changes)
- Firebase updated to `v11.13.0` -> `v11.14.0` (no api changes)
- Fyber updated `v8.3.6` to `v8.3.7` (no api changes) dkimitsa
- Tenjin SDK updated: `v1.14.9` -> `v1.14.10` (no api changes)


These pods were pushed to `https://central.sonatype.com/repository/maven-snapshots/` maven repo under `1.50.0-SNAPSHOTS` version.
[Source code @github](https://github.com/dkimitsa/robovm-robopods/tree/dev/v1.50.0)

Updates are not fully tested, please [open issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if bug found.
