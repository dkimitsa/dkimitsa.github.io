---
layout: post
title: 'Tracking slipped NPE, but in GAD v12.9.0'
tags: [signal, npe, robovm, bug, 'google-mobile-ads'']
---
`Tom-ski` have reported production level crash caused by `EXC_BAD_ACCESS` by java code inside try/catch block. It is well known issue of code like this:
```java
    public static void sayHello() {
        String s = null;
        try {
            s.toString();
        } catch (NullPointerException e) {
            e.printStackTrace();
        }
    }
```
caused by third party code that messes with `SIGSEGV` signal handler and makes it not accessible to RoboVM.
<!-- more -->

Any code that installs signal handlers should be called withing `Signals.installSignals` block: 
```java
Signals.installSignals(() -> {
    // initialize crash reporters here
}, true);
```
`installSignals` will restore RoboVM handler into chain after crashreportes installed own ones.. 

But `Tom-ski` case was valid and everything was set as expected. 

## Step1: Finding out signal handler 
Normally to have proper NPE handling, and keep handled NPE not being reported as crash by reporters, RoboVM always SHALL be active Signal handler. `signalHandler_npe_so_chaining` function in RoboVM case.   

As issue was not reproduced locally debug code was added to track active signal and report it to Firebase/Crashlytics in case it is not set to RoboVM one. A bit of debug code to get pointer/symbol name: 
```c
#include <robovm.h>
#include <signal.h>
#include <errno.h>
#include <dlfcn.h>
#include <stdio.h>

void* Java_org_robodebug_DebugTools_getSignalHandler(Env* env, Class* c, jint signum) {
    struct sigaction oldsa;
    int r = sigaction(signum, NULL, &oldsa);
    if (r < 0) {
       return NULL;
    } else if (oldsa.sa_flags & SA_SIGINFO) {
        return oldsa.sa_sigaction;
    } else if (oldsa.sa_handler != SIG_DFL) {
        return oldsa.sa_handler;
    } else {
   	return NULL;
    }
}

jint Java_org_robodebug_DebugTools_SIGBUS(Env* env, Class* c) {
    return SIGBUS;
}

jint Java_org_robodebug_DebugTools_SIGSEGV(Env* env, Class* c) {
    return SIGSEGV;
}

Object* Java_org_robodebug_DebugTools_dladdr_1sname(Env* env, Class* c, jlong handle) {
    Dl_info dlinfo;
    if (dladdr(LONG_TO_PTR(handle), &dlinfo) == 0 || dlinfo.dli_sname == NULL) {
        return NULL;
    }
    return rvmNewStringUTF(env, dlinfo.dli_sname, -1);
}

Object* Java_org_robodebug_DebugTools_dladdr_1fname(Env* env, Class* c, jlong handle) {
    Dl_info dlinfo;
    if (dladdr(LONG_TO_PTR(handle), &dlinfo) == 0 || dlinfo.dli_fname == NULL) {
        return NULL;
    }
    return rvmNewStringUTF(env, dlinfo.dli_fname, -1);
}
```

and Java bindings: 
```Java
package org.robodebug;

public final class DebugTools {
    private DebugTools() {
    }

    public static native int SIGBUS();
    public static native int SIGSEGV();
    public static native String dladdr_sname(long handle);
    public static native String dladdr_fname(long handle);
    public static native long getSignalHandler(int signo);
}
```

and Dumper code:
```Java
import org.robodebug.DebugTools;

public class Dumper {
    public static void dump(String title) {
        System.out.println(title);
        dumpSignal(DebugTools.SIGBUS());
        dumpSignal(DebugTools.SIGSEGV());
    }

    public static void dumpSignal(int signal) {
        long addr = DebugTools.getSignalHandler(signal);
        if (addr != 0)
            System.out.println(">>> " + signal + ": 0x" + Long.toHexString(addr) + DebugTools.dladdr_fname(addr) + " " + DebugTools.dladdr_sname(addr));
        else
            System.out.println(">>> " + signal + ": null");
    }
}
```

After deploying to prod and spending time needed to collect information, it was discovered that RoboVM handler was not top one but `GADRegisterSignalHandlers` (Google Mobile Ads).  

## Step 2. Observation and issue reproducing
`Tom-ski` doesn't use GAD directly but as mediator with different AD provider. This make it not 100% of time involved but only when mediator serves Google ads. 
Other moments were observed: 
- when linked statically GAD installs signal handler even if GAD is not used in app. Seems it utilizes `+ (void)load` or `__attribute__((constructor))` to get own code started.
- GAD seems to restore previous handler in chain after first hit (e.g. removes itself): `GADRegisterSignalHandlers` -> NPE in try/catch -> `signalHandler_npe_so_chaining`.

Simply code didn't reproduce the case. But stress test with multiple threads does. Minimal code to reproduce is following:
```java
Runnable NPE = () -> {
    try {
        ((String) null).equals("hello");
    } catch (NullPointerException ignored) {}
};
Thread t1 = new Thread(NPE);
Thread t2 = new Thread(NPE);
t1.start();
t2.start();
```

## Step 4. Why it is crashed
`Tom-ski` had run things under Xcode debugger and discovered that `GADRegisterSignalHandlers` installs handler with null actions, e.g.:
```c
struct sigaction sa = {0};
sigaction(SIGBUS, &sa, NULL);
```
and then calling default handler: `signal(SIGBUS, SIG_DFL);`

This scenario looks similar to restore and call previous handler in chain by why it is null? 

PS: XCode debugger breakpoint condition for `sigaction` symbolic breakpoint to catch the case is `($x1 != 0) && (*(void(**)(int))$x1 == 0)`;

## Step 5. Root case
It seems to be classic race condition case, analyzing normal and crash behaviour, `GADRegisterSignalHandlers` seems to work as following pseudo code:

```c
struct sigaction* saved_sa = {0};
void GADRegisterSignalHandlers() {
    struct sigaction sa;
    sa.sa_sigaction = &signalHandler;
    sigaction(SIGBUS, &sa, &saved_sa);     
}

static void signalHandler(int signum, siginfo_t* info, void* context) {
    // app crashed, end of world. restore previous handler, remove GAM from it
    sigaction(SIGBUS, &saved_sa, NULL);
    saved_sa = {0};
    // call previous handler 

    // do some reporting.
    signal(SIGBUS, SIG_DFL);
}
```

Failure scenario:
- thread1 and thread2 have both NPE at same time;
- signals are raised in both of these;
- first thread restores valid previous handler, removes GAM from handling and calls it;
- second thread processing signal as it was raised before removed by thread 1;
- second thread restores NULL handlers (e.g. not RoboVM ones) and call it;
- crash


## Bottom line and fix: 
Bad: 
- crash itself;
- the way `GADRegisterSignalHandlers` install it handler without being asked for (even if it doesn't crash, it will report handled in try/catch NPE as exception to GAM);

While it is up to Google to fix it, currently it might be handled by disabling GAM crash reporting with code as early as possible during app start-up:
```java
GADMobileAds.disableSDKCrashReporting() 
```
