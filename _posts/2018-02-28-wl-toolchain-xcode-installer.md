---
layout: post
title: "RoboVM for Windows/Linux: toolchain and iOS SDK installer"
tags: ["linux windows", idea]
---
First [POC Linux/Windows build]({{ site.baseurl }}{% post_url 2017-12-12-robovm-now-linux-windows %}) has major usability flaw: to start working on Linux/Windows user had to have access to Mac to obtain Xcode files and also  manually download and deploy toolchain files. Now it is solved with automatic downloader/installer:
![]({{ "/assets/2018/02/28/robovm-toolchain-setup.gif"}})  
This became possible with as result of following changes:  
<!-- more -->
- [Homemade iOS SDK]({{ site.baseurl }}{% post_url 2018-01-24-homemade-ios-sdk-for-linux-windows-robovm %}) -- no dependency on Apple;
- [Enhanced version check]({{ site.baseurl }}{% post_url 2018-02-21-wl-check-for-update-enh %});

Beside changes shown in gif above Update Checked was updated to monitor for upcoming updates to toolchain/sdk.
As per today these changes are not final and bugs are expected. All changes can be seen in github [dkimitsa/robovm](https://github.com/dkimitsa/robovm/commit/7efb7bbc9814c6e051109ee99ae6f35426989235) repo.
