---
layout: post
title: 'AltPods: Versioning changed, v1.56 released'
tags: ['bro-gen', binding, robopods, altpods]
---

There are two major updates regarding AltPods: 
- `v1.56.0` was released to Maven Central.
- The approach to versioning has changed. 

## Breaking Changes: Migration from bundle versions to individual pod versions 

**Previous drawbacks:**  
- A single version number was used for all modules within an `alt-pod` release. For example, `alt-pods` v1.56.0 contained `Firebase Crashlytics` v1.56.0, even though the actual bindings corresponded to `12.7.0`.
- All modules were released under the new `alt-pods` version, even if no changes were made to them. 
- Incompatible binding versions were sometimes grouped under a single `alt-pods` umbrella (e.g., an ads library might not be compatible with the Google Ads mediator if the latest version wasn't available yet).

**What has changed:**   
- Each module now has its own version built from the binary version. For example, if `SomeFramework` is `vA.B.C`, the pod version will be `vA.B.C.0`. The `.0` suffix allows for future patch versions.
- Each module is now independent and can be deployed separately. 
- There is no longer a need to deploy pods that haven't received updates.
- Snapshot versions will no longer be used for periodic updates. Only new frameworks currently under testing might be deployed as snapshots.

The artifacts for `v1.56.0` have already been deployed under their own independent versions to Maven Central and are ready for testing.  
The upcoming `v1.57.0` delivery is a work in progress, as it requires script adaptations, but it is expected to be available for testing later this week. 

[View the migration source code on GitHub](https://github.com/dkimitsa/robovm-robopods/pull/42)

Please [open an issue](https://github.com/dkimitsa/robovm-robopods/issues/new) if you encounter any bugs.
