---
layout: post
title: 'Libcore 10: Java Runtime from Android 10'
tags: [runtime, libcore10]
---

RoboVM is currently using runtime from Android 4.4 and it is being a pain for a long time due lack of Java 8 classes such as stream and outdated native components (like OpenSSL). Long term solution would be migration to JDK15+ but it would require updating compiler to understand Java language level 9+. For now biggest problem is DynamicInvoke support as RoboVM doesn't implement it and uses simplified RetroLambda approach. [Issue 514](https://github.com/MobiVM/robovm/issues/514) inspired to quick test a possibility. It was way simpler that migrate to JDK base and took only month to get working minimal vitable product.


## Libcore 10 overview
Highlights: 
- Same as Android 10 Runtime library;
- NIO2, Streams and other Java8 API;
- OpenSSL replaced with BoringSSL that enables TLS1.3;

## What is missing 
RoboVM native VM implementations (Android 4.4 based) and Android 10 have lot of differences and lot of things are not compatible. Quickes way was to minimize amount of changes in VM.  
Process was simplified by using RoboVM's core classes such as:
- java.lang.Class;
- java.lang.Object;
- java.lang.Thread;
- java.lang.ClassLoader;
- and few more (21 in total);

These classes might not provide recent Android 10 API and modern functionality (like compressed Strings). Probably each of these will be reviewed and reverted in future. 

## What is working
Currently it as tested with basic code for both Console MacOSX and iOS.  
Sample test code was deployed to repository [robovm-libcore10-smoketest](https://github.com/dkimitsa/robovm-libcore10-smoketest).

## Test builds and updates 
Code was modified a bit to not conflict with production builds:  
- version set to 10.0.xx;
- it will use `~/robovmx/cache` folder for produced .o files;

Builds will be deployed to release section of  [robovm-libcore10-smoketest](https://github.com/dkimitsa/robovm-libcore10-smoketest) repository. 
Please check that repository for future updates and fixes. 


There are several options to get test build:  
1. build from [source code](https://github.com/dkimitsa/robovm/tree/libcore10/robovmx)
2. Download pre-build Idea plugin from [Releases](https://github.com/dkimitsa/robovm-libcore10-smoketest/releases) section of smoke-test repository and install manually;
3. Setup `Custom plugin repository` as described by [link](https://www.jetbrains.com/help/idea/managing-plugins.html#repos):

* add `repository: https://dkimitsa.github.io/assets/updatePlugins.xml` as custom plugin repository in Idea, and then enter `` and install from there.
* in marketplace's search bar enter `repository: https://dkimitsa.github.io/assets/updatePlugins.xml`;
* install `RoboVM - Libcore 10` plugin from there.

## Issue reporting 
Please open the issue in the tracker of [robovm-libcore10-smoketest](https://github.com/dkimitsa/robovm-libcore10-smoketest/issues) repository;

