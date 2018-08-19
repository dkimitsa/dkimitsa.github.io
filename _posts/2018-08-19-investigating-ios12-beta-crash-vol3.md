---
layout: post
title: 'ios12 beta: root cause of RoboVM hangs/crashes vol3'
tags: [bug, gc, ios12]
---
This post continues series of investigations [#1]({{ site.baseurl }}{% post_url 2018-08-02-investigating-ios12-beta-crashes %}) [#2]({{ site.baseurl }}{% post_url 2018-08-08-investigating-ios12-beta-crash-vol2 %})). Root case -- boehm-gc doesn't resume all threads it paused during `GC_stop_world`. Details bellow.   
<!-- more -->
## GC_stop_world
During this calls GC stops everything, it obtains list of threads using `task_threads` then suspends every using `thread_suspend` and saves reference to this tread into `GC_mach_threads` array.

## GC_start_world
During this calls GC resumes everything, it obtains list of threads using `task_threads` then looks in `GC_mach_threads` for this thread record. And if it was suspended without issues -- resumes it.

## The problem
The issue here is that `GC_mach_threads` entry defined as:  
```c
struct GC_mach_thread {
    thread_act_t thread;
    GC_bool already_suspended;
  };
```  

And `GC_mach_thread.thread` is `mach_port_t` object and its value might not be same between `task_threads` calls. In other words during `GC_start_world` is not able find thread it suspended and not resuming it.

## Details
For more details I looked into [xnu-4570.41.2 sources @ apple opensource](https://opensource.apple.com/source/xnu/xnu-4570.41.2/). And threads that are returned by `task_threads` are `mach_port_t` which life cycle is reference count based. And once it is not reference anywhere it might be marked as not active and released.  
GC code does `mach_port_deallocate` to each thread reference but keeps this value.

I did quick POC code that just keeps calling `task_threads` twice and tries to compare list and issue was reproduced. I used `thread_info / THREAD_IDENTIFIER_INFO` to obtain system wide thread id to make sure that same thread exists but its mach_port id changed.

Obviously something changed in iOS12 as in previous versions this mach_port thread id wasn't changing (e.g. probably it was extra retained by system).

## Solution
Sadly today I have no time due vacation/national tournament to come with fix but hope to do it during next week.  
The solution: is not release mach_port thread id for threads marked as suspended.
