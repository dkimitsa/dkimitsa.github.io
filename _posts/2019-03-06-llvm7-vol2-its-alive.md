---
layout: post
title: 'LLVM7: vol2 -- its alive. Update#2'
tags: [llvm7, llvm]
---
# Update#2
As [per today](https://github.com/dkimitsa/robovm/commit/323fc12e02a4cf2402e16b1850436bc8721dd15f) code from [dkimitsa/robovm/llvm_70](https://github.com/dkimitsa/robovm/tree/llvm_70) is able compile and start application in simulator/device. It gets alive status.  What had been done:  
- RoboVm compiler infrastructure was updated to produce llvm 7 compatible IR code;
- DebugInformationPlugin was reworked to produce llvm7 debug information;
- Bunch of code was changed as it was broken as per llvm7 vision and produced crashes on asserts inside LLVM.

Next steps: 
- merge with [debugger_local_resolution](https://github.com/dkimitsa/robovm/commits/debugger_local_resolution) branch;
- resurect and run RoboVm tests;
- evaluate performance and size footprint of LLVM7 produced code; 
- migrate to LLVM 8 -- yep it has to be released this month.

# Previous updates:  
<!-- more -->
[Update#1 - kickoff]({{ site.baseurl }}{% post_url 2019-01-26-llvm7-vol1-kickoff %})
- [LLVM](https://github.com/llvm-mirror/llvm) and [Clang](https://github.com/llvm-mirror/clang) mirror repos are used as source base;
- CMake build files updated to compile version 7;
- Swig files for java bindings were cleaned up initially to produce clean results without need of post-work-arounds;
- LLMV Extra code -- set of helpers and wrappers used by RoboVM were updated to work with new LLVM;
- Swift files for bindings were updated to produce v7 wrappers;
- RoboVM compiler's code were updated as LLVM API was changed and lot of functionality was deprecated.
 
