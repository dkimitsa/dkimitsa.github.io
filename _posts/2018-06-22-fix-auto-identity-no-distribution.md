---
layout: post
title: 'bugfix: auto signature, auto profile was picking Distribution configuration, wrong'
tags: [fix]
---
[PR310](https://github.com/MobiVM/robovm/pull/310)  
Time to start 2.3.5 with fix. Fix to [own code that was intended to fix auto-signature pickup]({{ site.baseurl }}{% post_url 2018-05-16-fix-app-ext-autoprofile-bro-gen %}) ([PR293](https://github.com/MobiVM/robovm/pull/293)). It fixes a lot of stuff but introduced bug, that auto mode was picking distribution configuration (identity and profile). This will not work for debug/run configuration case. Fix is simple: consider only development identities:
```patch
--- Pair<SigningIdentity, ProvisioningProfile> pair = ProvisioningProfile.find(ProvisioningProfile.list(), SigningIdentity.list(), bundleId);
+++ Pair<SigningIdentity, ProvisioningProfile> pair = ProvisioningProfile.find(ProvisioningProfile.list(), SigningIdentity.list("/(?i)iPhone Developer|iOS Development/"), bundleId);

```
