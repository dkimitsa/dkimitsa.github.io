---
layout: post
title: 'not a bug: #235 NPE not handled when RoboVM used as SDK (framework target)'
tags: [fix, target-framework]
---
**History** [NPE not handled when Robovm used as SDK](https://github.com/MobiVM/robovm/issues/235)  

Few finding when working with RoboVM built framework from XCode. Today there are two examples listed of how to build framework with RoboVM:
<!-- more -->
1. [Java Framework Sample](https://github.com/robovm/robovm-samples/tree/master/MyJavaFramework)  -- to use framework do everything with tons JNI call;
2. [AnswerMe SDK Sample](https://github.com/robovm/robovm-samples/tree/master/AnswerMe) -- brings RoboVM development to next level and requires developer to write native code at Framework side.  

In both cases NPE inside try-catch will cause EXC_BAD_ACCESS during XCode run. Reason of this is that XCode is not allowing to bypass SIGSEGV SIGBUS signals. Entering following command when stopped at breakpoint just before test code has no effect (but should):
```
(lldb) process handle -p true -s false SIGSEGV SIGBUS
```

It is easy to check that RoboVM side is safe by just inspecting what signal handler is set with following code snippet and debug version of RoboVM runtime:
```objc
struct sigaction newSignalAction;
memset(&newSignalAction, 0, sizeof(newSignalAction));
sigaction(SIGSEGV, NULL, &newSignalAction);
```

if debug version of RoboVM is used it will contain pointer set to RoboVMs handler:
```
newSignalAction	sigaction
  __sigaction_u
    __sa_handler	void (*)(int)	(AnswerMeSDK`signalHandler_npe_so_nochaining at signal.c:333)  
    __sa_sigaction	void (*)(int, __siginfo *, void *)	(AnswerMeSDK`signalHandler_npe_so_nochaining at signal.c:333)
  sa_mask	sigset_t	0
  sa_flags	int	65
```

The only option is to run application without debugger attached. It is possible to be done by removing "Debug executable" flag in scheme settings (XCode menu: Product -> Scheme -> Edit Scheme -> Info Tab -> Uncheck "Debug executable"). Which means you can't debug while using framework that produces NPE at java side.

Corresponding [bug report at LLVM.org](https://bugs.llvm.org/show_bug.cgi?id=22868)
