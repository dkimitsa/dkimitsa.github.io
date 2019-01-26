---
layout: post
title: 'LLVM7: vol1 -- kickoff and Update#1'
tags: [llvm7, llvm]
---
Experiment with LLVM7 had been started. Current version of LLVM used is 3.6. Version 7 itself will not introduce any significant improvement for RoboVM compiler so far. Goal for this activity is to refresh dependencies code base.   
Source code of all changes is stored in separate branch [dkimitsa/robovm/llvm_70](https://github.com/dkimitsa/robovm/tree/llvm_70).   

# Update#1
Very brief: LLVM library code compiles but nothing was running yet.  

# What has been done so far:  
- [LLVM](https://github.com/llvm-mirror/llvm) and [Clang](https://github.com/llvm-mirror/clang) mirror repos are used as source base;
- CMake build files updated to compile version 7;
- Swig files for java bindings were cleaned up initially to produce clean results without need of post-work-arounds;
- LLMV Extra code -- set of helpers and wrappers used by RoboVM were updated to work with new LLVM;
- Swift files for bindings were updated to produce v7 wrappers;
- RoboVM compiler's code were updated as LLVM API was changed and lot of functionality was deprecated.

All these changes brought LLVM 7 code into project and now its possible to compile RoboVM compiler code.  
Things to do(high level overview):
- RoboVm has to produce LLVM7 IR code as 3.6 is not compatible anymore;
- Debugger DWARF IR information is also not compatible with 7.0 and has to be reworked;
- bugs-bugs-bugs         
  
