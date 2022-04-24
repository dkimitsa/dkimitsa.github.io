---
layout: post
title: 'Fix: wasteful memory usage by compiler'
tags: [fix, compiler, idea]
---
It is a 5-year-old [issue](https://github.com/MobiVM/robovm/issues/150) that was not bothering me much with my pet projects. Till I started playing with [something big](https://github.com/ergoplatform/ergo-wallet-app). 
In result with my default 4GB heap setting for Idea I was not able to complete a build due heap exhaustion:
![]({{ "/assets/2022/04/24/memory-usage-chart-4gb-before-fix.png"}})

Things got better with 8GB heap settings:
<!-- more -->
![]({{ "/assets/2022/04/24/memory-usage-chart-8gb-before-fix.png"}})

But even in this case after application was terminated, RoboVM owed about 3GB of memory.
Sequential launches were not increasing debt, and it seems to be a memory wasting than a leak. 

## long story short
After fixes applying following memory utilization was achieved:
![]({{ "/assets/2022/04/24/memory-usage-chart-4gb-after-fix.png"}})

- 4GB heap settings is enough to compile project;
- Most consuming moment was about 2.7 GB after garbage collector;
- when compilation is complete and app is started memory usage about 300MB (UI responsible, debugger happy);
- compilation time (31601 classes): reduced from `506.72 (8GB heap)` to `394.75 (4GB heap)` seconds.  

## the fix: post compilation pressure
Few things were just keeping not required any more data: 
- Soot was keeping all parsed classes in singleton. 
- Lot of structures were kept in Clazzes/Config that are not required after compilation (but were kept during application run).

Classes were extended with api to dispose data once it is not required. To maintain sanity and not use these classes when data is disposed the check was added the might throw `IllegalStateException` if class is misused after dispose. 

## the fix: in compilation pressure -- most memory consuming
This is the most affecting part. As during compilation of sample project heap utilization went up to 5.5GB. Root case of problem is same -- data was kept after it is not required anymore: 
- Soot -- once class is processed, no need to keep parsed methods' Body.
- ClazzInfo -- no need to keep dependency map for it once it processed through dependency trie.

## side way fix: string object number reduction
Compiler produces LLVM IR `.ll` file from Java class file. It's a text file and compiler converts instructions into strings and concatenates everything into one big text file. During operation, it produces thousands of string and glues them together. There was a [case]({{ site.baseurl }}{% post_url 2020-05-25-fix-oom-big-classes %}) recently when 4GB string was tried to be built. 
All these strings in results are being written into destination byte buffer using `Writer`. 
Optimization here is to make all entities to write itself directly to `Writer` instead of producing strings. 

## Code
The fix was proposed as [PR650](https://github.com/MobiVM/robovm/pull/650), [PR651](https://github.com/MobiVM/robovm/pull/651) 

Happy coding !
