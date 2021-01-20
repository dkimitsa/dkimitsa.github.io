---
layout: post
title: 're-thinking a way SWIFT dependencies are resolved'
tags: ['fix', swift]
---

## Background:
First time issue was reported [on gitter](https://gitter.im/MobiVM/robovm?at=5ff481b963fe0344963c53f6)
> libswiftCoreMIDI.dylib is not found in swift paths
>
Later it was communicated that workaround doesn't work during submition to Apple:
> ERROR ITMS-90209: "Invalid Segment Alignment. The app binary at 'SwiftSupport/iphoneos/libswiftCoreMIDI.dylib' does not have proper segment alignment. Try rebuilding the app with the latest Xcode version."

## Issues:
- RoboVM can detect swift dependencies only in dynamic frameworks. For static libraries/frameworks workaround [was introduced](https://github.com/MobiVM/robovm/pull/504).
- Today dynamic framworks might link again runtime swift libraries -- these should not be copied to `.app/Frameworks` folder. These dependecies can be observed using `otool -L` command:  

```
/usr/lib/swift/libswiftWebKit.dylib (compatibility version 1.0.0, current version 610.1.28, weak)
@rpath/libswiftAVFoundation.dylib (compatibility version 1.0.0, current version 1991.1.1, weak)  
```
there `libswiftWebKit.dylib` is from runtime location(and RoboVM can't copy it) and `libswiftAVFoundation.dylib` to be bundled into application (have to be bundled).

## Solutions:
<!-- more -->
1.
[Workaround](https://github.com/MobiVM/robovm/pull/504) doesn't work nice enough, and it is better to allow the linker to solve static dependencies. Moreover swift frameworks/libraries contain `LC_LINKER_OPTION` that specifies instructions how to link:
```
Load command 5
     cmd LC_LINKER_OPTION
 cmdsize 24
   count 1
  string #1 -lswiftCore
```

2.
Another fix: don't try to bundle swift frameworks that are available only runtime(e.g. `/usr/lib/swift/libswiftWebKit.dylib`). Existing `Pattern.compile("libswift.+\\.dylib");` was picking everything. Was changed to copy only one that are looked in `@rpath`: `Pattern.compile("@rpath/(libswift.+\\.dylib)");`

3.
In case deployment target is small to contain swift 5.1 linker will introduce `@rpath` dependencies in application binary itself. And these dependencies have to be bundled with application. So another step is added -- application binary is being scanned for Swift dependencies to be copied.

4.
[Workaround](https://github.com/MobiVM/robovm/pull/504) dropped, instead introduced config options:
```
    <swiftSupport>
        <swiftLibPaths>
           <path></path>
        </swiftLibPaths>
    </swiftSupport>
```
`swiftLibPaths` was moved there from the workaround and allows specifying custom swift location.  
`swiftSupport` once being added to config forces swift paths to be added to linker library search path and allows static linking with swift. 

## Code 
Changes proposed as [PR552](https://github.com/MobiVM/robovm/pull/552)
