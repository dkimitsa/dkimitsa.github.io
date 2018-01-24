---
layout: post
title: 'Homemade iOS SDK for Linux/Windows port of RoboVM'
tags: ["linux windows", hacking]
---
Apple doesn't allow SDK/Development Tools to be used on non-apple built hardware. This was limiting RoboVM usage on Linux/Windows platforms as it was required to manually copy files from Mac (breaking the license). Ok then, lets make own SDK for RoboVM.
<!-- more -->
## What problem does it solve:
- as it own made SDK has nothing to do anymore with Apple license;
- now binary can be build on Linux/Windows;
- it can be distributed;
- now it is possible to create automatically downloadable package from Idea and install platform tools right from configuration dialog without manual operations.

## Still there are limitations:
- still missing DeveloperDiskImage.dmg. And this can't be solved by any 'homebrew' alternative. So run and debug is complicated.

## But hey not so bad:
- even if run/debug not possible application will be deployed to device and can be started manually by tapping the icon;
- if you run Linux/Windows on Apple HW you can copy these files from Mac;
- if you debug any app on your device in Xcode/RoboVM then you have `DeveloperDiskImage.dmg` mounted. And you can run and debug on this device from Linux/Windows. Till next time device is rebooted.

## Preview version of SDK
[Sample version](https://goo.gl/1njuUs) is available for download

## How this is possible:
Lucky for us Apple uses .tbl stub files instead library binaries for a while:
> TAPI is a __T__ext-based __A__pplication __P__rogramming __I__nterface. It replaces the Mach-O Dynamic Library Stub files in Apple's SDKs to reduce SDK size even further.
>
> The text-based dynamic library stub file format (.tbd) is a human readable and editable YAML text file. The TAPI projects uses the LLVM YAML parser to read those files and provides this functionality to the linker as a dynamic library.

### Short story:
All that is required is to get export symbol information and generate own .tbd files. Apple pack all dylibs and frameworks into single file `dyld shared cache`. I've already touched [this topic]({{ site.baseurl }}{% post_url 2017-12-29-ios-dyld-sharedcache %}). There is bunch of implementations in the wild which work with cache and produces .tbd files but it is always better to have own implementation that depends on abandoned projects. There is an implementation from [Apple dev]( https://github.com/ributzka/tapi) and I was able to run it but I didn't enjoy the path required for it.

### Long story
There is lot of Mach-o parsing involved. Dive into [project on github](https://github.com/dkimitsa/robovm-sdk-builder).
Following links are very helpful:
- [Apple implementation of generator]( https://github.com/ributzka/tapi) - abandoned;
- [dsc_extractor (from apple)](http://opensource.apple.com/source/dyld/)
