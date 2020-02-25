---
layout: post
title: "W/L: Device system log and screenshoot capture for Idea (same as in Xcode)"
tags: ["linux windows", idea]
date: 2018-04-27 10:00:00
---
Continue extending functionality for Linux/Windows port. Today device system log and screenshot capture were added. This functionality is available in Xcode on Mac but on Windows/Linux it would require external tool. It was required to adjust [libmobiledevice](http://www.libimobiledevice.org) bindings, some RoboVM classes and Idea plugin to make it available for user.   
**Whats new in idea**
*Syslog* functionality almost copies Android Logcat as code from AOSP was used as reference.   
![]({{ "/assets/2018/04/27/robo-device-log.png"}})  

<!-- more -->
*Capture screenshot* icon is available on Syslog panel and can be seen on screenshot above.
*Syslog* panel can be activated from both `View-Tool Windows` and `RoboVM` menus.
![]({{ "/assets/2018/04/27/robo-device-log-menus.png"}})  

**Source code**
All changes can be seen in the [wl_device_support branch](https://github.com/dkimitsa/robovm/tree/lw-device-syslog) on project github page. `libmobiledevice` binaries are not committed and has to be built from source.

**Changes to libmobiledevice**
Both syslog and screenshot are possible to retrieve by using [libmobiledevice](http://www.libimobiledevice.org). This library is already used in RoboVM for application deployment to device. Most if the work that were done is to create bindings for `syslog_relay` and `screenshotr` apis. It was done extending `swig` script and using `swig`.
Beside binding itself there was middle level helpers introduced to make work with `libmobiledevice` more code friendly.
