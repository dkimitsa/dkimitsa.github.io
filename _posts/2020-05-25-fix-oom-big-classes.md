---
layout: post
title: 'RoboVM compiler: out of memory -- huge class case'
tags: [fix]
---
RoboVM compiler is quite memory hungry by its nature: creates and manipulates a bunch of strings, have to keep parsed classes structures while compiling etc. Also there is a bug opened [#150](https://github.com/MobiVM/robovm/issues/150). Usually its enough to give JVM bigger heap (`-Xmx4g`) to make everyone happy.  
There [was a report](https://gitter.im/MobiVM/robovm?at=5e871b26a95bc942564df0a5) on gitter channel about `OOM` that happens on a single file that can't be fixed by increasing heap size. File just contained about 8K static final string fields. 

To reproduce the case dummy file with 8000 strings was generated:  
```java
public class DBkeys
{
    static public final String T0001 = "";
    static public final String T0002 = "";
    // lot of more these 
    static public final String T8000 = "";
}
```

Compilation using default heap settings just produce `Out of memory exception` but once JVM granted plenty of ram (`-Xmx32g`) it ends up with exception:
> java.lang.OutOfMemoryError: Required array length too large

RoboVM went beyond of JVM limits trying to allocate byte array with size longer than JVM supported (`Integer.MAX_VALUE`  - 2 gig). In other words -- giving all memory you have to JVM heap will not solve this issue. 

## Root case
OOM happens when RoboVM compiles java code into LLVM IR code. This constant string only class doesn't fit in MAX possible 2 GB java string. Let's see what is in there.  
<!-- more -->

Long story short: to access field RoboVM calculate offset of the field from start of the object. Class/Object structure is defined in ClassType structure. This structure is being generated while compiling class.  
Compiler insert offset calculation code in form of `offset = {raw structure}, filed#no`. Each time. For such big class (with 8000 fields) this structure will be 170000 chars long. Compiler uses it three times per fields. 

Lets do a math:  
> 8000 fields * 170000 * 3 = 4.080.000.000 bytes. 

So 4Gb just to store only structure without actual code. 

## Fix  
Typedef it once, use alias:  
> %ClassType = type { very-very-very long struct}   
> offset = %ClassType, filed#no  

Much better. Result: .ll file is only 16Mb (vs 4Gb+ before the fix)

Fix was delivered as [PR485](https://github.com/MobiVM/robovm/pull/485).
