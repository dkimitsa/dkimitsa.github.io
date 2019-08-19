---
layout: post
title: 'Ios13: Fixing broken launch on simulator target'
tags: [simulator, fix]
---
With `Xcode 11` RoboVm can't deploy to simulator anymore. It fails with:
> simlauncher[70781:19762861] *** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '+[DVTSimulatorApplication simulatorApplicationForDevice:]: unrecognized selector sent to class 0x1077a3590'        

`simlauncher` was borrowed from `MOE` project and internaly build around tonns of hacks, and its hard to keep maintain it this way.  
Meanwhile Apple provides `simctl` tool allong with `Xcode` and its functionality quite enough for RoboVm needs. 

[PR402](https://github.com/MobiVM/robovm/pull/402) deliver migration to `simctl` and introduces following changes: 
- rmeoves `simlauncher`;
- boots simulator if required with `simctl boot`;
- deploys application with `simctl install`;
- and launches it with `simctl launch`.

It works for me with Xcode11 and also has to be backward compatible.