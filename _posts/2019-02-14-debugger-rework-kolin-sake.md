---
layout: post
title: 'Debugger: improving kotlin support and fixing #220'
tags: [debugger, kotlin, jvm]
---
This post continues series of debugger rework, refer to [previous post]({{ site.baseurl }}{% post_url 2018-07-06-debugger-kotlin-smap-support %}). Today changes are focused around [#220 Exception in Lambda transformation using modified soot](https://github.com/MobiVM/robovm/issues/220):  
<!-- more -->
## Root case 
Soot was crashing as it was not able to solve local variables type constraints as same variable was assigned with diffirent type values. E.g. with `integer` and `Object`.  
Soot performs local split run on each local assignment so each assignment produces new variable and with uniq value and type. Then locals variables of same type are get packet to revert split and minimize variable count.  
As result Soot not always can pack splited locals into one variable. This was a problem for debugger before [variable resolution rework]({{ site.baseurl }}{% post_url 2018-06-27-debugger-variable-resolution-rework %}).  

## What wrong with kotlin, again
[#220](https://github.com/MobiVM/robovm/issues/220) was crashing on following case:  
```
class: kotlinx.coroutines.scheduling.WorkQueue: 
method: kotlinx.coroutines.scheduling.Task pollExternal$default(kotlinx.coroutines.scheduling.WorkQueue,kotlin.jvm.functions.Function1,int,java.lang.Object)>

0 = {JIdentityStmt@1008} "l0: index=0 := @parameter0: kotlinx.coroutines.scheduling.WorkQueue"
1 = {JIdentityStmt@1009} "l1: index=1 := @parameter1: kotlin.jvm.functions.Function1"
2 = {JIdentityStmt@1010} "l2: index=2 := @parameter2: int"
3 = {JIdentityStmt@1011} "l3: index=3 := @parameter3: java.lang.Object"
4 = {JAssignStmt@1012} "$stack0#2: = l2:, index=2 & 1"
5 = {JIfStmt@1013} "if $stack0#2: == 0 goto l3:, index=3 = l0:, index=0.<kotlinx.coroutines.scheduling.WorkQueue: int consumerIndex>"
6 = {JAssignStmt@1014} "$stack0#3: = <kotlinx.coroutines.scheduling.WorkQueue$pollExternal$1: kotlinx.coroutines.scheduling.WorkQueue$pollExternal$1 INSTANCE>"
7 = {JAssignStmt@1015} "l1:, index=1 = (kotlin.jvm.functions.Function1) $stack0#3:"
8 = {JAssignStmt@1016} "l3:, index=3 = l0:, index=0.<kotlinx.coroutines.scheduling.WorkQueue: int consumerIndex>"
...
```

Problem here is that `l3: index=3` is being assigned with `java.lang.Object` value(line 3) and `int` value(line 8). This happend as variable was not split in two locals.   
Why? RoboVM was preventing spliting variables that were passed as parameter(here is slot 3 is a parameter).  
Why? Java compiler reserved these slots for parameters and it was one time during entire method space so RoboVM was minimizing split effect by forbiding splitting slot variables used by parameters.   
But kotlin reuses these slots for local variables. 

*So parameter slots are also has to be allowed for split*. 

## The fix
Is allow to split parameter slots. These changes are applied to [soot branch](https://github.com/dkimitsa/soot/commit/a0ffbeb18e4b5bb7d4b4fa43143647d1594cf77e). Also RoboVM debugger code [was changed](https://github.com/dkimitsa/robovm/commit/511aaac7368d49ce5885709e3935fbb205416117) to allow the fact that parameter variable is not persistent during method.

## Code
All changes are present in [debugger_local_resolution](https://github.com/dkimitsa/robovm/tree/debugger_local_resolution) branch.  Also [PR349](https://github.com/MobiVM/robovm/pull/349) was created.

## Idea build for testing 
test build for 2.3.6-SNAPSHOT available at [google drive](https://goo.gl/j7ZC27)