---
layout: post
title: 'Linux/Windows WIP: not a bug - "Error in compatibility flow" exception'
tags: ["linux windows", hacking, xib2nib]
---
While working on Linux/Windows port and improving xib2nib faced following issue that was happening only when custom build toolchain was used:
```txt
*** Assertion failure in -[UIView _nsis_center:bounds:inEngine:forLayoutGuide:], /BuildRoot/Library/Caches/com.apple.xbs/Sources/UIKit/UIKit-3698.33.7/NSLayoutConstraint_UIKitAdditions.m:3347
*** Terminating app due to uncaught exception 'NSInternalInconsistencyException', reason: 'Error in compatibility flow'
```

It was happening when using recently added UIKit functionality in xibs (UILayoutGuide in this case). Long investigation to simple solution is bellow:
<!-- more -->

## Decompiling UIKit framework
Exception message doesn't tell much of what is wrong and what to do. And there is not much mention in internet about this case. Source code is not available then decompiled version of UIKit will work. Evaluation version of [Hopper Disassembler](https://www.hopperapp.com) is good enough to find routine that makes assertion fails: `_UILayoutEngineSolutionIsInRationalEdgesConsultingDelegate`.
It performs validation as in following simplified pseudo code:
```c
bool _UILayoutEngineSolutionIsInRationalEdgesConsultingDelegate() {
    return _UIApplicationLinkedOnOrAfter(0x80000);
}    

bool _UIApplicationLinkedOnOrAfter(uint32_t v) {
    return UIApplicationLinkedOnVersion >= v;
}
```

In general it crashes if version stored in `UIApplicationLinkedOnVersion` variable is older that `8.0.0`.

## Finding out the version
First thing is to read value of `_UIApplicationLinkedOnVersion`. Running `original` toolchain in Xcode:
```objc
extern uint32_t _UIApplicationLinkedOnVersion;
NSLog(@"UIApplicationLinkedOnVersion %d", _UIApplicationLinkedOnVersion);
```
Gives `11.2.0` (0x0b0200)

Running same in RoboVM with Linux/Windows toolchain:
```java
@Library(Library.INTERNAL)
@CustomClass("PieChartViewController")
public class PieChartViewController extends UIViewController implements ChartViewDelegate {
    @GlobalValue(symbol="_UIApplicationLinkedOnVersion", optional=true)
    public static native int UIApplicationLinkedOnVersion();

    @Override
    public void viewDidLoad() {
        super.viewDidLoad();

        System.out.printl("" + UIApplicationLinkedOnVersion());
    }
}
```

Gives '7.0.0' -- Oh!

## Where 7.0.0 comes from
After making simple test with `working` set of assets/resources it turned out that only executable itself matters. Fast check for mach-o load command (LC_VERSION_MIN_IPHONEOS) shows following for `official` executable :
```txt
> otool -l Main | grep -A 4 LC_VERSION_MIN_IPHONEOS

Load command 9
      cmd LC_VERSION_MIN_IPHONEOS
  cmdsize 16
  version 11.0
      sdk 11.2
```
and for Linux/Windows toolchain:
```txt
Load command 9
      cmd LC_VERSION_MIN_IPHONEOS
  cmdsize 16
  version 11.0
      sdk n/a
```

thats it, `linker` doesn't fill SDK version attribute.

## False lead: TAPI MC/MachObjectWriter
First finding lead in TAPI library was very promising - [strange code snippet](https://github.com/tpoechtrager/apple-libtapi/blob/5dc2fda8884ca6a96dd7f81c40349d318ae5c1b5/src/apple-llvm/src/lib/MC/MachObjectWriter.cpp#L813):
```cpp
// Write out the deployment target information, if it's available.
if (VersionInfo.Major != 0) {
  assert(VersionInfo.Update < 256 && "unencodable update target version");
  assert(VersionInfo.Minor < 256 && "unencodable minor target version");
  assert(VersionInfo.Major < 65536 && "unencodable major target version");
  uint32_t EncodedVersion = VersionInfo.Update | (VersionInfo.Minor << 8) |
    (VersionInfo.Major << 16);
  MachO::LoadCommandType LCType;
  switch (VersionInfo.Kind) {
  ...
  case MCVM_IOSVersionMin:
    LCType = MachO::LC_VERSION_MIN_IPHONEOS;
    break;
  }
  write32(LCType);
  write32(sizeof(MachO::version_min_command));
  write32(EncodedVersion);
  write32(0);         // reserved.
}
```

In this snipped SDK version is never get filled and being written with ZERO (// reserved). Very similar to `sdk n/a` above. But it is not case.

## not a bug -- linker did what it can
Turned out that there is undocumented (`man ld` doesn't show it) linker command that specifies SDK version to be embedded: '-sdk_version <version>'. Adding it to linker invocation fixed the issue.

## Why `official` toolchain works
If version is not specified linker tries to get it from `sysroot` parameter, [here is the code](https://github.com/tpoechtrager/cctools-port/blob/e527b6f87f0613de7ec6d214f81d41fb5621a5b0/cctools/ld64/src/ld/Options.cpp#L5092):
```c
// if -sdk_version not on command line, infer from -syslibroot
if ( (fSDKVersion == 0) && (fSDKPaths.size() > 0) ) {
    const char* sdkPath = fSDKPaths.front();
    const char* end = &sdkPath[strlen(sdkPath)-1];
    while ( !isdigit(*end) && (end > sdkPath) )
        --end;
    const char* start = end-1;
    while ( (isdigit(*start) || (*start == '.')) && (start > sdkPath))
        --start;
    char sdkVersionStr[32];
    int len = end-start+1;
    if ( len > 2 ) {
        strlcpy(sdkVersionStr, start+1, len);
        fSDKVersion = parseVersionNumber32(sdkVersionStr);
    }
}
```

If `official` toolchains are used (normal MacOS operations) RoboVM finds path to SDK's sysroot as bellow:
> /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS11.2.sdk

but in RoboVM/Linux it is simplified to following (e.g. without version):
> ~/.robovm/platform/Xcode.app/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk
