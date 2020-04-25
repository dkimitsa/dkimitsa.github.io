---
layout: post
title: 'Debugger: maintenance 2020.4'
tags: [fix, debugger, idea]
---
[PR475](https://github.com/MobiVM/robovm/pull/475) delivers an amount of fixes related to the debugger:
## fixed: debugger crashes when setting breakpoints in specific method
*Root case* is changes in previous [improvement: debugger reports now only line number where breakpoint can be hit]({{ site.baseurl }}{% post_url 2020-02-06-debugger-maintenance %}).  
Improvement was to re-use breakpoint map structure as `lines available for breakpoints`. Debugger resolves that map when line map is being requested.  
If all lines in the method are available for breakpoints and number of lines is even to eight this results in `available for breakpoints map` completely filled with zeroes. Linker puts zero filled memory structures to [__bss](https://en.wikipedia.org/wiki/.bss) data segment. Debugger was not expecting symbols in this segment and was crashing with `unable to resolve symbol` exception.

*The fix*: refactored buffer reader and added ZeroReader to support `.bss` data segment.       

## fixed: broken stepping over line that loads inner class
this happens when stepping over code similar to bellow:
```kotlin
inner class Test2()
fun test() {
    print(1)
    Test2()    // stepping over 
    print(2)   // will not stop here first time     
}
```
 
Root case: is `class loaded` event from device/simulator that suspends thread. Once thread is suspended all stepping constraints removed. Debugger after receiving `class loaded` even tells the target to resume. But there is no stepping constraint available anymore.   
*The fix*: restore stepping constraint in device on thread resume if stepping is active.

## fixed: stepping inside kotlin lambda  
The issue can be observed in code similar to bellow: 
```kotlin
      fun test() {
/*7*/    list.forEachIndexed { index, str ->
/*8*/       text = str + index       // <-- will not step or stop at this line 
/*9*/    }
     }
```
Kotlin injects collections code to user class under extended line number range and uses [SMAP]({{ site.baseurl }}{% post_url 2018-07-06-debugger-kotlin-smap-support %}) to map extended range into class line number range. Example above turned into following pseudo code after SMAP un-mapping:  
```kotlin
    fun test() {
/*7*/   val l = list as Iterable<String>; var idx = 0;
/*8*/   val it = l.iterator(); while(it.hasNext()) { val str = it.next(); text = str + idx++ }
    }
```
As result there is multiple operation an line 8. but only one breakpoint/stop point will be instrumented before first operation in line. This results in breakpoint/stop point injected before `l.iterator()` and no one inside the loop. 

*The fix*: inject stop/breakpoint hooks before SMAP line numbers unmapped and code get collapsed in single line.

## optimizations: application binary stripped 
Binary was not stripped before as `strip` removes local symbols as result `debugger` was not able to resolve required symbols. 
To support stripping its requires to pre-save symbols debugger depends on. This was done by putting these into exported symbols list.    
Having binary stripped results in following (sample data based on empty project, x86_64 simualator):
- application binary size reduced: from 113M -> 45M, faster deployment;
- number of symbols debugger have to parse reduced: from 317K -> 28K, faster parsing(10x time) and debugger startup, smaller memory footprint as no need to keep never used ~300K symbols (about 60M);

## rework: refactored buffer reader 
Code to manipulate memory buffer refactored to provide abstract interface instead of ByteBuffer centric implementation. Code refactored to use interfaces that allows to provide implementation with different data source. For example `NullReader` to simulate reading from `.bss` segment. 
Also, it allows to have future improvements required for windows builds (workaround for mapped file issues). 

## rework: .spfpoffset value fetched during compilation (was runtime)
Initially it was mistake to make `.spfpoffset` symbols global as it resulted in many "symbol already defines" error.  
Root case of it is LLVM emitter who adds `.spfpoffset` for all functions (even always inline or private). As result there was many private symbols with same name turned into global.  
Situation was fixed (for having binary stip sake) by reworking the way .spfpoffset retrieved:    
was:
- symbols were referenced in `llvm.used` just to survive dead code elimination;
- debugger resolved `.spfpoffset` runtime and fetched value;  

now:
- during compilation `.spfpoffset` is being fetched from object file;
- value added to class debug information;
- `.spfpoffset` is not required anymore and can be eliminated as not used.

## other changes: dependency order changed
Debugger module reworked to be independed from LLVM/Compiler module. 
Now its stand-alone module. Instead following was changed:
- `compiler` module has direct dependency to `debugger`;
- `idea`/`eclipse`/`maven`/`gradle` plugins don't depend anymore on `debugger` (as it is part of compiler now);
