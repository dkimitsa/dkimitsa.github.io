---
layout: post
title: 'ios12 beta: why RoboVM hangs/crashes synthetic case in iOS12'
tags: [bug, gc, ios12]
---
**UPDATE**: there is a [follow up]({{ site.baseurl }}{% post_url 2018-08-19-investigating-ios12-beta-crash-vol3 %}) on this topic.  

Few days ago I narrowed [issue](https://github.com/MobiVM/robovm/issues/317) to simple case (check [this post]({{ site.baseurl }}{% post_url 2018-08-02-investigating-ios12-beta-crashes %})) but it didn't answer why this was happening. Today I had time to digg this rock and find pice of code that cause issue. What has been done:  
<!-- more -->
# boehm-gc code was updated
[boehm-gc](https://github.com/dkimitsa/bdwgc) was forked and updated today to [ivmai/bdwgc](https://github.com/ivmai/bdwgc) master while preserving RoboVM related changes. Sample was running on new code base but issue still was there.

# Investigating boehm-gc and root case
It was enough to run `System.gc()` flow with debugger to find out problematic place. Root case code is [GC_stop_world](https://github.com/dkimitsa/bdwgc/blob/f105edbf7420e92734d61bdc4d6b6b734dfe63a4/darwin_stop_world.c#L560) routine in `darwin_stop_world.c`.  
This routines stops all threads before performing garbage collection. The moment here that it obtains all task threads using [task_threads](https://developer.apple.com/documentation/kernel/1537751-task_threads?language=objc) call. It includes all application threads not only RoboVM ones. In empty iOS12 project I can see 4 threads running, including main, event loop and other system added threads. Suspending and resuming these threads in combination with suspending/resuming application causes iOS app to halt and crash. Probably it is Apple's bug as it crashes due internal object access synchronization and time limited operations with grand central dispatch.

# Reproducing with native XCode project
start/stop world was extracted from `boehm-gc` and put into simple iOS app to have proof of concept. It is available in my [codesnippets repo](https://github.com/dkimitsa/codesnippets/tree/master/ios12bc-case-native/ios12_hang_demo). And reproduce exactly same case.

# Handling the case
**WARNING**: shall not be used in any case but POC. Check next paragraph for details.  
Simple way to handle it is to tell `boehm-gc` to stop/resume only threads it knows about. This can be done by adding `--disable-threads-discovery` parameter to configuration script when building `RT`. And it DOES solve the issue. You can pick up pre-build ARM64 [libgc.a](https://github.com/dkimitsa/codesnippets/raw/master/ios12bc-case-native/ios12_hang_demo/lib/libgc.a) for testing. Replace on in `~/.robovm-sdks/robovm-2.3.5-SNAPSHOT/lib/vm/ios/arm64` folder.

# BUT
Limiting GC to handle threads it knows is **DANGEROUS**. As once java code is called from thread GC doesn't know about (e.g. created at native side) most likely will cause crash as GC will not able to stop that thread during gc(). There is a nice text in [README.darwin](https://github.com/dkimitsa/bdwgc/blob/master/doc/README.darwin) about this case:
>Since not all uses of the GC enable clients to override pthread_create()
before threads have been created, the code for stopping the world has
been rewritten to look for threads using Mach kernel calls. Each
thread identified in this way is suspended and resumed as above. In
addition, since Mach kernel threads do not contain pointers to their
stacks, a stack-walking function has been written to find the stack
limits. Given an initial stack pointer (for the current thread, a
pointer to a stack-allocated local variable will do; for a non-active
thread, we grab the value of register 1 (on PowerPC)), it
will walk the PPC Mach-O-ABI compliant stack chain until it reaches the
top of the stack. This appears to work correctly for GCC-compiled C,
C++, Objective-C, and Objective-C++ code, as well as for Java
programs that use JNI. If you run code that does not follow the stack
layout or stack pointer conventions laid out in the PPC Mach-O ABI,
then this will likely crash the garbage collector.

# Bottom line
RoboVM is not a root case. Memory leaks are not root case. GC is not root case (directly). Even considering GC plays quite rude and stops entire world hangs are happening due Apple's logic and best case scenario is Apple to fix it.

The is no solution yet as today I just focused on finding root case and hope to have solution in next days. Stay tuned. Happy coding.
