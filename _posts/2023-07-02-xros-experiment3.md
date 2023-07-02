---
layout: post
title: 'Experiment 3: XrOS (Vision OS) support'
tags: [apple, xros, visionos, technote, llvm, robovmx]
---
Experiment 3 of RoboVMx started to play with additional platforms (such as TvOS). There was an interest for use RoboVM with XrOS project in [gitter](https://matrix.to/#/!mqUEluorTjXZPncMfe:gitter.im/$aPrr1CbDPGU6hTaU1YK4UM8bpDq06gEe4iTSXDHzOSQ?via=gitter.im&via=matrix.org) recently and this looks like a good point to kick off this experiment.   
VisionOS SKS are mostly swift based and this a problem to provide bindings to RoboVM. But anyway its good point to just get RoboVM Java VM code running in RxOS environment. 
And first stage is to allow RoboVM to be available in Xcode project as framework.
Experiment is focused first to 
## How to use
<!-- more -->
Sadly at moment of writing there was no `Xcode15 Beta2` Github Action runners available. 
Way to test it is to build it from source of this branch [RoboVMX:experiments/experiment/3-additional-platforms](https://github.com/robovmx/robovmx/tree/experiment/3-additional-platforms).
- Short story: run `build.sh` in root of source tree.
- Long story: check [MobiVM/Wiki](https://github.com/MobiVM/robovm/wiki/Developer-Guide) for details.

To produce Framework for XrOS, `robovm.xml` should be adapted in following places:
```
<config>
    <!-- The framework targets XrOS. -->
    <os>xros</os>
    <arch>arm64</arch>
    <arch>arm64-simulator</arch>
    <target>xcframework</target>
    
    ....
    
</config>    
```

## Technical details 
Same as in [Experiment#1: m1 support]({{ site.baseurl }}{% post_url 2021-05-26-native-m1-support %}) binary has to specify valid XrOS platform type in MachO/LC_BUILD_VERSION load command. 
It has to be in both: VM runtime binaries and object files produced from Java class files.  

###  Compiling VM libs for XrOS
These steps are very common as with adding m1, to compile for XrOS changes related to adding two targets to CMakeFile:
- "arm64-apple-xros${XROS_MIN_VERSION}-simulator" 
- "arm64-apple-xros${XROS_MIN_VERSION}"

Xcode 15.0 Beta2 with VisionOS runtime installed required to compile.
All changes are in this [commit](https://github.com/robovmx/robovmx/commit/4d832b5bc048cc44863b5e94f994afda14d5e9d6).

### Adapting LLVM
8 year old LLVM v3.6 knows nothing about XrOS. Will add changes there with patch. Lucky for me, LC_BUILD_VERSION support already was added there during m1 support integration. 
Changes related to add `XrOS` constant and related code to parse `arm64-apple-xros1.0-simulator` target and produce platform type (11, 12) in LC_BUILD_VERSION. 
Confusing things -- these changes are not public yet, so had to introduce own name for these constants:   
```
enum PlatformType {
    PLATFORM_MACOS = 1,
    PLATFORM_IOS = 2,
    PLATFORM_TVOS = 3,
    PLATFORM_WATCHOS = 4,
    PLATFORM_BRIDGEOS = 5,
    PLATFORM_MACCATALYST = 6,
    PLATFORM_IOSSIMULATOR = 7,
    PLATFORM_TVOSSIMULATOR = 8,
    PLATFORM_WATCHOSSIMULATOR = 9,
    PLATFORM_DRIVERKIT = 10,
   
    PLATFORM_XROS = 11,
    PLATFORM_XROSSIMULATOR = 12
 };
```

All changes are in this [commit](https://github.com/robovmx/robovmx/commit/6922ead69c83cc35e8c126081ce8d56b72ee9114?diff=split).

### RoboVM compiler adaptation 
Nothing special, just adding `OS.xros` enum entry and fixing everything around to get Framework built.
All changes are in this [commit](https://github.com/robovmx/robovmx/commit/7a29cf927ba33f643ead375050b01d51436792dd).

### Bonus
As a bonus, Framework target now is not limited to iOS only and can produce MacOS/XrOS binaries. 

### Code available on github as [RoboVMx/Experiment3](https://github.com/robovmx/robovmx/tree/experiment/3-additional-platforms)

## Happy coding!
Please report any issue to [tracker](https://github.com/robovmx/robovmx/issues/new).
