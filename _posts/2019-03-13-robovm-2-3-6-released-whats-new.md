---
layout: post
title: 'RoboVM 2.3.6 released, what''s new'
tags: [whatsnew]
---

## What's new
- [fix] Xcode project generator: added argument type to IBAction [#323](https://github.com/MobiVM/robovm/pull/323)
- [added] forceLinkMethods configuration option to preserve method from aggressive tree shaker, [details](https://dkimitsa.github.io/2018/08/27/aggressive-treeshaker-workarounds/)
- [fixed] `compile` dependency declaration bug with gradle 4;
- [added] putByte/getByte support for Unsage
- [updated] cocoa touch bindings up to ios 12.1, [new frameworks etc]({{ site.baseurl }}{% post_url 2018-11-09-ios-12-1-binding %})
- [added] experimental option to enable incremental compilation, thanks @dthomes. [#334](https://github.com/MobiVM/robovm/pull/334)
- [fixed] support for vectors which unchainder ARKit, [details]({{ site.baseurl }}{% post_url 2018-12-11-vector-data-types %})   
- [fixed] `Invalid Swift Support` issue when submitting to apple build created using XCode 10.1/10.2;
- [added] preservePorts option to Sygnals.installSignals API, to supress reporting of handled NPE to Crashlitics, [details]({{ site.baseurl }}{% post_url 2019-02-21-mach-exception-handler-and-crashlytics %})   

## Useful tutorials
- [Tutorial: using app-extensions in RoboVM iOS application]({{ site.baseurl }}{% post_url 2018-01-18-tutorial-using-app-extensions-with-robovm %})
- [Tutorial: writing Framework using improved framework target]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %})
- [Tutorial: Crash Reporters and java exceptions]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %})
- [Tutorial: Investigating silent crash of Release build (e.g. Port death watcher fired)]({{ site.baseurl }}{% post_url 2018-02-12-investigating-silent-crash %})
- [Tutorial: Interface Builder integration in RoboVM]({{ site.baseurl }}{% post_url 2017-10-24-xcode-ib %})
- [Tutorial: using bro-gen to quick generate bindings]({{ site.baseurl }}{% post_url 2017-10-19-bro-gen-tutorial %})
