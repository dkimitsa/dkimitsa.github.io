---
layout: post
title: 'ios12 beta: investigating hang up and crash'
tags: [bug, gc, ios12]
---
**UPDATE**: there is a [follow up]({{ site.baseurl }}{% post_url 2018-08-08-investigating-ios12-beta-crash-vol2 %}) on this topic.  
**UPDATE2**: there is a [follow up]({{ site.baseurl }}{% post_url 2018-08-19-investigating-ios12-beta-crash-vol3 %}) on this topic.  

libgdx developers got concerned about crashes in ios12 beta and there is an [issue about it](https://github.com/MobiVM/robovm/issues/317). I was not able to reproduce it using steps provided by [Eric Nondahl](https://github.com/ericnondahl/libgdx-ios12-sample) as was trying to reproduce it on empty iOS system, and following actions helped much:
* switching to other apps will help;
* it is better other apps to be OGL ones;

These steps resulted in reproducing Eric's POC. But [POC](https://github.com/ericnondahl/libgdx-ios12-sample)) is quite verbose as packed with different cases:  
<!-- more -->
* libgdx;
* bulk memory allocations;
* mixing libgdx with native UI (eg opening/closing dialogs).

Crash log for this case looks as bellow:
```
Exception Type:  EXC_CRASH (SIGKILL)
Exception Codes: 0x0000000000000000, 0x0000000000000000
Exception Note:  EXC_CORPSE_NOTIFY
Termination Reason: Namespace SPRINGBOARD, Code 0x8badf00d
Termination Description: SPRINGBOARD, scene-update watchdog transgression: com.mycompany.myapp exhausted real (wall clock) time allowance of 10.00 seconds | ProcessVisibility: Foreground | ProcessState: Running | WatchdogEvent: scene-update | WatchdogVisibility: Foreground | WatchdogCPUStatistics: ( | "Elapsed total CPU time (seconds): 5.290 (user 5.290, system 0.000), 26% CPU", | "Elapsed application CPU time (seconds): 0.006, 0% CPU" | )
Triggered by Thread:  0

Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x000000023c630e78 semaphore_wait_trap + 8
1   libdispatch.dylib             	0x000000023c47fe58 _dispatch_sema4_wait$VARIANT$mp + 24
2   libdispatch.dylib             	0x000000023c480900 _dispatch_semaphore_wait_slow + 136
3   FrontBoardServices            	0x000000023f50b3ac -[FBSSceneSnapshotRequestHandle performRequestForScene:] + 516
4   FrontBoardServices            	0x000000023f511fb4 -[FBSSceneSnapshotAction snapshotRequest:performWithContext:] + 244
5   FrontBoardServices            	0x000000023f4cac48 -[FBSSceneSnapshotRequest performSnapshotWithContext:] + 376
```

As per [Understanding and Analyzing Application Crash Reports](https://developer.apple.com/library/archive/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-APPINFO) `Code 0x8badf00d` explained as:
>The exception code 0x8badf00d indicates that an application has been terminated by iOS
because a watchdog timeout occurred. The application took too long to launch, terminate,
or respond to system events. One common cause of this is doing synchronous networking
on the main thread. Whatever operation is on Thread 0 needs to be moved to a background
thread, or processed differently, so that it does not block the main thread.

Once reproduced I was able to minimize POC to smallest problematic code: `System.gc()`.  
Libgdx is not required here to reproduce. Problem can be reproduced on basic `RoboVM iOS app without storyboard` template by extending it with following code:
```java
public class MyViewController extends UIViewController implements CADisplayLink.OnUpdateListener {
    ...
    private CADisplayLink displayLink;

    public MyViewController() {
        this.displayLink = new CADisplayLink(this);
        this.displayLink.addToRunLoop(NSRunLoop.getMain(), NSRunLoopMode.Default);
        ....
    }

    long ts = System.currentTimeMillis();
    @Override
    public void onUpdate(CADisplayLink displayLink) {
        System.gc();
        if (System.currentTimeMillis() - ts > 1000) {
            ts = System.currentTimeMillis();
            label.setText("onUpdate. " + (++clickCount);
        }
    }
}
```

## Bottom line
Most likely there is a problem in [boehm-gc](https://github.com/robovm/bdwgc) and next steps would be to run it with debug information enabled and analyze it, other options would be upgrading to up today version of `bdwgc` as `RoboVM` uses quite outdated one (2012).  
Work in progress. Stay tuned.
