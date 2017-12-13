---
layout: post
title: "RoboVM for Windows/Linux: sources code release"
tags: ["linux windows", "release"]
---
Source codes for RoboVM part and toolchain has been released:

- Modified RoboVM [github](https://github.com/dkimitsa/robovm/tree/linuxwindows)
- Toolchain [github](https://github.com/dkimitsa/robovm-toolchain)

Project depends on external projects that were forked and modified:
- cctools/ld64 fork [github](https://github.com/dkimitsa/cctools-port/tree/mingw)
- Microsoft/WinObjC fork [github](https://github.com/dkimitsa/WinObjC/tree/xib2nib)

Project that are used without changes on repo level (but patched during build):
- Facebook/Xcbuild [github](https://github.com/facebook/xcbuild)
