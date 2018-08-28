---
layout: post
title: 'ios12 fix delivery to boehm-gc upstream'
tags: [gc, ios12]
---

[The fix]({{ site.baseurl }}{% post_url 2018-08-27-aggressive-treeshaker-workarounds %}) is confirmed by users.  
Same fixes were merged into [boehm-gc](https://github.com/dkimitsa/bdwgc) code base and delivered as [PR232](https://github.com/ivmai/bdwgc/pull/232), also [issue #231](https://github.com/ivmai/bdwgc/issues/231) was created to track case.  
Also up-to-date `boehm-gc` with these changes for RoboVM is available in my branch [dkimitsa/boehm-gc](https://github.com/dkimitsa/bdwgc). Will investigate if there is any benefit to upgrade to recent `boehm-gc` once ios12 is released.
