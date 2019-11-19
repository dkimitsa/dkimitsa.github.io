---
layout: post
title: 'RoboVM 2.3.8 released, what''s new'
tags: [whatsnew]
---

## What's new
- [fixed] [#408](https://github.com/MobiVM/robovm/issues/408) Cannot compile AUMIDIEvent;
- [fixed] [#414](https://github.com/MobiVM/robovm/issues/414) Deployment to ios13 device, updated libimobiledevice lib's bindings;
- [fixed] swiched to `simctl` to support Xcode11 simulator [PR#410](https://github.com/MobiVM/robovm/pull/410);
- [improvement] kotlin improvements: debugger and compilation [PR#349](https://github.com/MobiVM/robovm/pull/349)
- [fixed] [#387](https://github.com/MobiVM/robovm/issues/387) CGBitmapContext.create fails on real device for big sizes;
- [added] support for packed structures [PR#378](https://github.com/MobiVM/robovm/pull/378);
- [fixed] Broken enum marshallers cleanup [PR#377](https://github.com/MobiVM/robovm/pull/377);
- [fixed] Debugger crashed when native thread exited while paused on breakpoint [PR#384](https://github.com/MobiVM/robovm/pull/384);

## What's in progress 
Meanwhile work in progress happening in [support for JDK12 tools on host](https://github.com/MobiVM/robovm/tree/jdk12) branch, it contains following changes: 
- all builds scripts reworked to allow building RoboVM using JDK9+ (works with JDK13) [PR#400](https://github.com/MobiVM/robovm/pull/400);
- ios13 bindings [PR#406](https://github.com/MobiVM/robovm/pull/400);
- Idea plugin workaround for long-going `android gradle` faset [#242](https://github.com/MobiVM/robovm/issues/242), [PR#401](https://github.com/MobiVM/robovm/pull/401);
- On-going series of [Cocoa13-postfixes]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})

Happy coding!  
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).