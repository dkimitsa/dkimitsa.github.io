---
layout: post
title: 'ios12 beta crash: fix for testing [vol4]'
tags: [bug, gc, ios12]
---

This post continues series of investigations [#1]({{ site.baseurl }}{% post_url 2018-08-02-investigating-ios12-beta-crashes %}) [#2]({{ site.baseurl }}{% post_url 2018-08-08-investigating-ios12-beta-crash-vol2 %}) [#3]({{ site.baseurl }}{% post_url 2018-08-19-investigating-ios12-beta-crash-vol3 %}). It delivers fix described in previous post: stop/start world changed in way to not loos mach port reference to thread between stop and start world calls. For fix I've used old code `boehm-gc` that is used in [MobiVM](https://github.com/MobiVM/bdwgc) and not recent one I've updated recently just to minimize amount of new problem introduced. I'll will be back to updated code in one of next snapshots once iOS12 issue is confirmed.  

I've created a [PR1](https://github.com/MobiVM/bdwgc/pull/1) where all changes can be evaluated.  
Pre-built library for testing is also available [lib_fix_candidate/libgc.a](https://github.com/dkimitsa/codesnippets/raw/master/ios12bc-case-native/ios12_hang_demo/lib_fix_candidate/libgc.a)

Please test and report any issues to [gitter channel](https://gitter.im/MobiVM/robovm)
