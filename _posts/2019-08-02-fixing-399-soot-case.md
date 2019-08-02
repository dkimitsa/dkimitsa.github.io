---
layout: post
title: 'Fix #399: ArrayIndexOutOfBounds inside soot'
tags: [fix, soot]
---

Its investigation in context of [issue #399](https://github.com/MobiVM/robovm/issues/399). Soot was crashing on big class files with exception:  
> java.lang.ArrayIndexOutOfBoundsException: -29458
	at soot.coffi.CFG.processCPEntry(CFG.java:2652)

Investigation shown following byte code that Soot was trying to parse: 
>    Code:
      stack=4, locals=0, args_size=0
         0: ldc2_w        #36078              // double 0.017453292519943295d

Easy check in hex presentation (`-29458=0xFFFF8CEE ` and `36078=0x8CEE`) shows that its classical signed/unsigned misstake.  
Root case was in `Instruction_intindex` where argument (the index) was read as `singed short` which was ok till the value fit positive boundary of short, but turned into negative value once overflows. This class is super class for following set of Java bytecode which were affected as well (corresponding to JVM spec):  
* anewarray
* checkcast
* getfield
* getstatic
* instanceof
* invokedynamic
* invokeinterface
* invokenonvirtual
* invokestatic
* invokevirtual
* ldc2
* ldc2w
* multianewarray
* new
* putfield
* putstatic

Fix is trivial: [read value as unsigned short](https://github.com/dkimitsa/soot/commit/2f0e189600f49b85355864ae5afa475dfa085486)

NP: this bug was hidding from very beginning and was just waiting big enough class file to detonate. 
