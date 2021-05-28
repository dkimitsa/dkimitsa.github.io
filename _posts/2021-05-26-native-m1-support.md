---
layout: post
title: 'Native Apple Silicon M1 support'
tags: [apple, m1, technote, llvm]
---
Apple Silicon proposed in [PR586](https://github.com/MobiVM/robovm/pull/586). It includes: 
- `m1` `arm64` version of `llvm`, `ilibmobiledevice`, `hfsconmpressor` libraries to allow it being used with arm64 version of java;
- support for `arm64` iOS simulator and `arm64` MacOSX console target. 

How fast it is (compiling classes using 8 threads): 
> (Mac Mini m1):   Compiled 3317 classes in 36.45 seconds  
> (Mac Pro W3565): Compiled 3317 classes in 111.60 seconds  
> (MacBook Pro, i5-4278U @ 4 threads): Compiled 3317 classes in 219.09 seconds  

## How to use   
<!-- more -->
Pre-build binaries were deployed as `10.1.1-SNAPSHOT` to [com.robovmx](https://github.com/robovmx) fork and accessible for testing using [Idea plugin](https://github.com/robovmx/robovm/releases/tag/exp1%2Fbuild1) or with gradle:
```groovy
buildscript {
  repositories {
    mavenLocal()
    mavenCentral()
    maven { url 'https://oss.sonatype.org/content/repositories/snapshots' }
  }
  dependencies {
    classpath 'com.robovmx:robovm-gradle-plugin:10.1.1-SNAPSHOT'
  }
}

apply plugin: 'java'
apply plugin: 'robovm'
sourceCompatibility = 1.8
targetCompatibility = 1.8

repositories {
  mavenLocal()
  mavenCentral()
  maven { url 'https://oss.sonatype.org/content/repositories/snapshots' }
}

ext {
  roboVMVersion = "10.1.1-SNAPSHOT"
}

robovm {
}

dependencies {
  compile "com.robovmx:robovm-rt:${roboVMVersion}"
  compile "com.robovmx:robovm-cocoatouch:${roboVMVersion}"
  testCompile "junit:junit:4.12"
}
```

# Technical details 

## Compiling lib for m1 arm64
### CMake pain 
While running CMake on Apple platform it adds `arch` and `sysroot` parameters to compiler flags. However, there are [CMAKE_OSX_SYSROOT](https://cmake.org/cmake/help/latest/variable/CMAKE_OSX_SYSROOT.html) and [CMAKE_OSX_DEPLOYMENT_TARGET](https://cmake.org/cmake/help/latest/variable/CMAKE_OSX_DEPLOYMENT_TARGET.html#variable:CMAKE_OSX_DEPLOYMENT_TARGET) to control the beast, but it still doesn't allow you to control completely build flags.  
The only way to make it do what expected is to tell CMake we are doing cross compilation by specifying `CMAKE_SYSTEM_NAME`.  
In this case it will try to provide `SYSROOT` but it can be controlled by overriding `CMAKE_OSX_SYSROOT`.

### Adapting LLVM
[commit](https://github.com/dkimitsa/robovm/commit/eb6d1f772c3fd3f79d52b79365e71d574963cc23)  
Once `m1` introduced `arch64` binary can be any of the following `arm64` platforms: `MacOSX on m1`, `ios on device`, `ios on m1 simulator` (with tvos it ever worse). A try to link `arm64` files produced by RoboVM result in error:  
> building for iOS Simulator, but linking in object file built for iOS, for architecture arm64

To differentiate platforms Apple now uses `LC_BUILD_VERSION` mach-o load command:  
```
struct build_version_command {
    uint32_t cmd;      // LC_BUILD_VERSION
    uint32_t cmdsize;  // sizeof(struct build_version_command) +
                       // ntools * sizeof(struct build_tool_version)
    uint32_t platform; // platform
    uint32_t minos;    // X.Y.Z is encoded in nibbles xxxx.yy.zz
    uint32_t sdk;      // X.Y.Z is encoded in nibbles xxxx.yy.zz
    uint32_t ntools;   // number of tool entries following this
};

// Values for platform field in build_version_command.
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
 };
```

`LC_BUILD_VERSION` replaced `LC_VERSION_MIN_IPHONEOS`. There is no support for this command in LLVM3.6 used with RoboVM. All required parts of code were borrowed from [LLVM12](https://github.com/llvm/llvm-project/tree/main/llvm) and added as [a patch](https://github.com/dkimitsa/robovm/commit/eb6d1f772c3fd3f79d52b79365e71d574963cc23). 

#### Fun fact 1
`LLVM12` is not able to compile an apple asm file due missing corresponding branches in [bool DarwinAsmParser::parseBuildVersion](https://github.com/llvm/llvm-project/blob/632ebc4ab4374e53fce1ec870465c587e0a33668/llvm/lib/MC/MCParser/DarwinAsmParser.cpp#L1161):  
> org.robovm.llvm.LlvmException: com.android.okhttp.HttpHandler:2:17: error: unknown platform name
>    .build_version iossimulator, 14, 0

These were added to [patch](https://github.com/dkimitsa/robovm/blob/eb6d1f772c3fd3f79d52b79365e71d574963cc23/compiler/llvm/patches/06-emit-build-version.patch#L543);

#### Fun fact 2 
Clang 3.6` is so old so [doesn't expect](https://github.com/dkimitsa/robovm/blob/eb6d1f772c3fd3f79d52b79365e71d574963cc23/compiler/llvm/patches/06-emit-build-version.patch#L1052) iOS version greater than 9.99;

### Compiling ilibmobiledevice
[commit](https://github.com/dkimitsa/robovm/commit/3fdeac26ef3ac83821c92b67c607d6743119fd50)  
Where was only cross compilation issue same as in case of LLVM. 
NOTE: I was not able to test it with device as for development I rent m1 mac in cloud without ability to attach device to it.

### Compiling libhfscompressor
[commit](https://github.com/dkimitsa/robovm/commit/ef2db150d11848aaf9cab81555bf07038e14e0a8)
Sources for this lib was missing in the project. Picked from https://github.com/robovm/hfscompressor, adapted to build using CMake and to use recent dependencies.  
Added option to compile for both `macosx-arm64` and `macosx-x86_64`.  `HfsCompressor.java` updated to load platform specific binary

## RoboVM compiler adaptation
[commit](https://github.com/dkimitsa/robovm/commit/f933271699e1e3b998b5061b3e40e2a36d1108a0)  
This commit adds `Environment (Native/Simulator)` option that allows to make difference between targets of same arch (e.g. iOS arm64 vs MacOSX arm64).  
For example build for `arm64` now could be targeted to `ios-arm64`, `macosx-arm64`, `ios-arm64-simulator`.
LLVM detects proper target from the triple, and it is being generated now with respect to `Environment` option.
Also linker `-miphoneos-version-min=` parameter was replaced with `--target=` one that accepts the triple.

Changes were done to folder names: environment is being attached to every arch name where applicable. For example:
```
vm/lib/arm64/
vm/lib/arm64-simulator
vm/lib/thumbv7
vm/lib/x86-simulator
vm/lib/x86_64-simulator
```
All code that used `os` + `arch` as key for platform now uses `os` + `arch` + `env`.
 
## VM Native libraries adaptation
[commit](https://github.com/dkimitsa/robovm/commit/ed91bf0eb27dd66f2a252ddd40b4d2163b994b4e)
As linker now depends on `LC_BUILD_VERSION` separate set of VM native libraries were compiled for `macosx-arm64`, `ios-arm64-simulator` targets.

## Simulator pickup logic adapted
[commit](https://github.com/dkimitsa/robovm/commit/a81360c08ac07af7bef5314baad9aeb250eb4e6e)
Logic adapted to following:
- simulator considered to have `arm64` arch only on m1 host and if simulator's version is 14+
- x86 simulator is not available on `arm64` m1 host

## Intellij Idea plugin changes
[commit](https://github.com/dkimitsa/robovm/commit/46b421f684b88a1464eae6df57796aed1e3a3b97)
- `auto` simulator arch is set to the on of the host. E.g. on m1 will be set to `arm64`
- console target now has `arch` option that allows to select `x86_64`/`arm64` arch on m1 silicon;  

![]({{ "/assets/2021/05/28/m1_console_run_config.png"}})

## Other fixes
[commit](https://github.com/dkimitsa/robovm/commit/1ebe5e2f7cf66a015402ae6e6c418aa22d367273)
These issues were discovered during run against debug version of llvm lib and crashes happened on assertions:
* llvm: changed Load/Store atomic access ordering from `unordered` to `monotonic` as it was crashing during AtomicExpandPass for x86 target. Line of code where it crashed `getStrongestFailureOrdering`. In release build were assertions removed it falls down into `Monotonic` case anyway.
* bug fixed: wrong calculation of struct offset for Vectorized Structs. Root case: vector was wrapped into single element struct like `{<2 x float>}` and any try to get offset for member 1+ returns garbage (or crash on assert in debug version of llvm). Workaround -- getting element storage size and multiply by index

## Bonus track: github action workflows to build native libraries
[commit](https://github.com/dkimitsa/robovm/commit/e4b7d8bdcb14d7aa3100770854e241c3dad394fa)
While `macos-latest` is still pointing to MacOSX 10.15 workflows are not able to build m1 `macosx-arm64` target. `x86_64` can be build today and `arm64` will be available once `macosx-11` is opened to public.
Worflows allow to build `libhfs`, `libmobiledevice`, `libllvm` natives for MacOSX. In case of success build new branch with artifacts is pushed. Then it can be merged into master.  
Build are triggered manually and are parametrized:
- "Arguments to build.sh" -- specifies arguments to build.sh script, e.g. target to build ;
- "Target branch to push artifacts" -- name of the branch were artifacts will be pushed;
- "Commit message" -- message to be used while committing artifacts;

![]({{ "/assets/2021/05/28/github_build_natives_action.png"}})


## Happy coding!  
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).
