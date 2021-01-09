---
layout: post
title: 'CocoaTouch: regenerated to fix static methods issue'
tags: ['bro-gen']
---
It is a follow up of previous post: [bro-gen: static methods and polymorphism]({{ site.baseurl }}{% post_url 2020-11-30-bro-gen-static-methods-and-polymorphism %}).
CocoaTouch was regenerated with updated `bro-gen` and it caused 1000+ changes in source code. There are massive `supportsSecureCoding()`, `getLayerClass()` and also it exposed not available before constructors and APIs.

The changes proposed as [PR543](https://github.com/MobiVM/robovm/pull/543) and targeted to 2.3.13 version of RoboVM. 
