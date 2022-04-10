---
layout: post
title: 'Fix: TimeoutException $classname.finalize() timed out after 10 seconds'
tags: [fix]
---

This issue is rare and affects most user. It can be seen once registered [for default Java exception handler]({{ site.baseurl }}{% post_url ./2020-07-08-firebase-and-java-stacktraces %}).  
Scenario for this crash is following:  
- iOS application goes to background/suspend (just enough to minimize it);
- resumed after > 10s.

Crash is rare but more user you have more crashes you will see.

## Root case
[FinalizerWatchdogDaemon](https://github.com/MobiVM/robovm/tree/master/compiler/rt/libcore/libdvm/src/main/java/java/lang/Daemons.java#L202) looks after `FinalizerDaemon` and responsible to terminate App if object spends too much time in `.finalize()` call.
Scenario is following:
- `FinalizerDaemon` picks object to dispose and calls `.finalize()` to allow object performing clean-ups;
- `FinalizerWatchdogDaemon` detects it and startes looking for it;
- Application is minimized/suspended;
- all threads are suspended (including Daemons);
- (time passed, 10+ seconds);
- Application is resumed;
- `.finalize()` is still busy;
- `FinalizerWatchdogDaemon` has already counted 10+ seconds its being observing class doing finalization and terminates app, considering that finalizer is stuck. 

## Trying to reproduce 
<!-- more -->
Bug is easy to reproduce -- just need to delay finalize, and minimize app before `sleep` ends.    
```c
    private void test() {
        new Tst();
        System.gc();
        System.gc();
        System.gc();
        System.gc();
        System.gc();
    }

    static class Tst {
        @Override
        protected void finalize() throws Throwable {
            System.out.println("Going to sleep, minimize app!");
            Thread.sleep(5000);
            System.out.println("Should not get there!");
            super.finalize();
        }
    }
```

# The fix
Proper fix would be to know exact time application spends active/suspended but its platform dependant.
Another workaround would play here: divide single 10s naps of `FinalizerWatchdogDaemon` into two 5s ones.  
It will not keep precise timing of 10s and have corner cases (like if finalizer takes more than 5s and suspend is happen) but in most cases this will solve the issue. 

# Code 
The fix was proposed as [PR643](https://github.com/MobiVM/robovm/pull/643)
