---
layout: post
title: 'BuildType options for Framework target'
tags: ['target-framework', robovmx]
---
RoboVM application compiled into Framework has known issues running with debugger attached in XCode. 
In particular, it suffers from NullPointerException that might happen in Java code, even in try/catch section:
```java
    @Method(selector = "hello")
    public static void sayHello() {
        String s = null;
        try {
            s.toString();
        } catch (NullPointerException e) {
            e.printStackTrace();
        }
    }
```
will cause:
![]({{ "/assets/2023/07/05/xcode-EXC_BAD_ACCESS.png"}})

RoboVM handles this case by intercepting `SIGSEGV` signal. But GDB intercepts `EXC_BAD_ACCESS` and doesn't allow it to go System level where it causes `SIGSEGV`.
To co-work with GDB RoboVM includes `robovm-debug` module that is being linked for Run configuration and does as written in comments:  
<!-- more -->
```c
// We only want the code in here to be linked into debug builds since some of the code below
// calls private functions (exc_sever).

/*
 * Install a mach exception handler which intercepts EXC_BAD_ACCESS and prevents GDB from
 * seeing it. If we don't do this GDB will not pass the EXC_BAD_ACCESS along to the OS so
 * that it can be converted into SIGBUS/SIGSEGV and handled by our signal handler. Instead
 * the EXC_BAD_ACCESS will just be raised again and again and again...
 * The code here has been inspired by the GC's code in os_dep.c and the mailing list post at
 * http://lists.apple.com/archives/darwin-dev/2006/Oct/msg00122.html.
 */
```

## Step1: adding `robovm-debug` to framework target 
To work with Xcode -- framework has to be linked with debug module. `Create Framework` dialog was extended with Build Type picker:
![]({{ "/assets/2023/07/05/idea-new-framework-dialog.png"}})

* Release -- build for production/submitting to AppStore as doesn't contain private API usage;
* Debug -- debug version of framework. Also includes `robovm-debug`;
* Release with debugger support -- same as Release but allow debugging in Xcode (includes `robovm-debug`)

## Step2: Xcode GDB configuration 
Running framework with included `robovm-debug` cause different kind of trouble:
![]({{ "/assets/2023/07/05/xcode-SIGSEGV.png"}})

`EXC_BAD_ACCESS` is now converted into `SIGSEGV` as was promised. But Xcode is still not usable. 
We have to configure XCode to don't stop on Signals that RoboVM handles: `SIGSEGV`, `SIGBUS`. For this following to be entered into `lldb` prompt when application is paused on Breakpoint (!):
```
 process handle --pass true --stop false --notify true SIGSEGV  
 process handle --pass true --stop false --notify true SIGBUS  
```

There is a way to make these steps automatic on app launch:
- create symbolic breakpoint for UIApplicationMain;
- add two `Debugger command` actions to configure signals;
- set "Automatically continue after evaluating actions" to resume app automatically.

![]({{ "/assets/2023/07/05/xcode-breakpoint-with-settings.png"}})

In result test code should produce expected stacktrace to log:
![]({{ "/assets/2023/07/05/xcode-proper-log.png"}})

### Code 
Is available on github as [RoboVMx/Experiment3/PR19](https://github.com/robovmx/robovmx/pull/19/)

## Bonus track -- weak symbols mess 
`robovm-debug` is activated by invoking `registerDarwinExceptionHandler` from `signal.c` where its weak implementation is present:
```c
// Weak stub for the function in vm/debug/src/debug.c. If librobovm-debug.a isn't
// linked in this version will be used and won't do anything.
void registerDarwinExceptionHandler(void) __attribute__ ((weak));
void registerDarwinExceptionHandler(void) {
}
 
 jboolean rvmInitSignals(Env* env) {
    throwableInitMethod = rvmGetClassMethod(env, java_lang_Throwable, "init", "(Ljava/lang/Throwable;J)V");
    if (!throwableInitMethod) return FALSE;
    if (sem_init(&dumpThreadStackTraceCallSemaphore, 0, 0) != 0) {
        return FALSE;
    }
    if (!installNoChainingSignals(env)) {
        return FALSE;
    }
#if defined(DARWIN)
    registerDarwinExceptionHandler();
#endif
    return TRUE;
}
```

Strong implementation is in `debug.c`.

### Problem -- concept is broken
Probably in old versions of Clang/GCC it was different but today Clang doesn't care about weak flag during resolution of symbol (it will just not produce duplicate symbol error).
First object file matching the symbol will be used as target. The best moment here -- it start searching from module (source file) where invocation is happening. In our case we had both declaration of weak stub and invocation in same module.
As result replacement with Strong implementation wasn't working. 

Solution:
- leave only forward declaration in `signal.c`;
- move weak stub to standalone module `signal-darwin.c`;
- make sure we link `robovm-debug` before `robovm-core` (it contains `signal.c`, `signal-darwin.c`) so Strong implementation will be resolved instead of Weak stub.

### nm -- lies
Doesn't show symbol as weak:
> nm ./librobovm-core-dbg.a | grep registerDarwinExceptionHandler
> 0000000000000000 T _registerDarwinExceptionHandler

But if check symbol table of it:
```
sym = "_registerDarwinExceptionHandler"
nlist = {NList@861}
  n_strx = 596
  n_type = 15
  n_sect = 1
  n_desc = 128
  n_value = 0
```

it seems to be correct (n_desc = 128):  
```
/*
* The N_WEAK_DEF bit of the n_desc field indicates to the static and dynamic
* linkers that the symbol definition is weak, allowing a non-weak symbol to
* also be used which causes the weak definition to be discared.  Currently this
* is only supported for symbols in coalesed sections.
  */
  #define N_WEAK_DEF	0x0080 /* coalesed symbol is a weak definition */
```

But shows different result if bitcode is embedded:
> nm ./librobovm-core-dbg.a | grep registerDarwinExceptionHandler  
> ---------------- W _registerDarwinExceptionHandler 



