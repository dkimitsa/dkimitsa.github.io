---
layout: post
title: 'Tracking slipped NPE, but in Crashlytics v11.15.0'
tags: [signal, npe, robovm, bug, 'crashlytics']
---

Few months since last [Tracking slipped NPE]({{ site.baseurl }}{% post_url 2025-08-16-signal-mess-npe-gam %}) and here we go again:
- same user reported;
- same scenario (code that handles a lot of NPE in try/catch);
- this time with Crashlytics.

Time for dissection... 
<!-- more -->

## TLTR (spoiler)
- Crashlytics installs signals in background thread, so `Signal.install()` doesn't preserve proper chain;
- Due to ^^^^ it will report NPE inside try/catch as crash (and will not report crash after it);
- Its Mach exception handler is not protected against simultaneous events, and will crash if this happens;
- Its Signal handler is not protected against simultaneous events, will mess signal handler, and crash.

## Environment 
Well known and simple scenario:
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

And well know setup code: 
```java
Signals.installSignals(() -> {
    FIRApp.configure();
}, true);
```

## Case 1: Signal handler
Once `Signals.installSignals` is called with `preservePorts = true`, all Mach handlers installed inside lambda are rolled back (sometimes, but details bellow). 
In this case Signal handler code will handle NPE, and its code available in [FIRCLSSignal.c](https://github.com/firebase/firebase-ios-sdk/blob/11.14.0/Crashlytics/Crashlytics/Handlers/FIRCLSSignal.c).  
It does:  
- FIRCLSSignalHandler - handler itself 
- --> FIRCLSSignalSafeRemoveHandlers -- replaces signal handlers with `SIG_DFL`; 
- --> FIRCLSSignalRecordSignal -- saves crash;
- --> FIRCLSSignalSafeInstallPreexistingHandlers:
- --> ---> again calls FIRCLSSignalSafeRemoveHandlers
- --> ---> restores previous signal handlers;

What wrong here: 
- there is no multi-thread lock on handling signals, can happen simultaneous;
- non-atomic and messy restore to previous signal chain: `SIG_DFL` first, then previous;

And there is a crash in case of simultaneous NPE events, log looks as bellow: 
```
FIRCLSSignalHandler 11
FIRCLSSignalHandler 11
FIRCLSSignalSafeRemoveHandlers 11 
FIRCLSSignalSafeRemoveHandlers 11 
FIRCLSSignalRecordSignal
FIRCLSSignalRecordSignal
FIRCLSSignalSafeInstallPreexistingHandlers 11
<crash>
```

and crash is inside `FIRCLSSignalRecordSignal` as it is surprised by simultaneous nature of event and just kills the app (!!!!):
```objc
static void FIRCLSSignalRecordSignal(int savedErrno, siginfo_t *info, void *uapVoid) {
  if (FIRCLSContextMarkAndCheckIfCrashed()) {
    FIRCLSSDKLog("Error: aborting signal handler because crash has already occurred");
    exit(1);       /// <<<<<< CRASH HERE 
    return;
  }
}  
```

## Case 2: Mach handler
Once `Signals.installSignals` is called with `preservePorts = false` Crashlytics will handle everything in Mach exception handler that has higher priority than signals [FIRCLSMachException.c](https://github.com/firebase/firebase-ios-sdk/blob/11.14.0/Crashlytics/Crashlytics/Handlers/FIRCLSMachException.c).  
Due to nature of Mach message handling it is handled in one thread one-by-one and simultaneous mess as in case of `FIRCLSSignalHandler` but again there is no protection against simultaneous event:  
- FIRCLSMachExceptionServer - runs message receive loop;
- FIRCLSMachExceptionDispatchMessage - handles received message;
- --> FIRCLSMachExceptionUnregister - unregisters from next messages;
- --> FIRCLSSignalSafeInstallPreexistingHandlers - restores signals (same non-atomic as above);
- --> FIRCLSMachExceptionRecord - records crash.

Whats wrong here: 
- simultaneous event will produce multiple messages that will be sent to server; 
- server process these in single thread so no run condition will occur;
- by handling first message it unsubscribes from future messages due events;
- (and not expect anymore messages)
- as simultaneous happened before unsubscription it will be delivered to server. 
- server will process unexpected second event and crash.

Crash happens in similar code due unexpected conditions (and it just kills the app): 
```objc
static bool FIRCLSMachExceptionRecord(FIRCLSMachExceptionReadContext* context,
                                      MachExceptionMessage* message) {
  ...
  if (FIRCLSContextMarkAndCheckIfCrashed()) {
    FIRCLSSDKLog("Error: aborting mach exception handler because crash has already occurred\n");
    exit(1);       /// <<<<<< CRASH HERE
    return false;
  }
  ...
}
```

## Case 3: Why at all it handles JAVAs NPE as RoboVM should be the one ?
Following setup should restore RoboVMs power:
```java
Signals.installSignals(() -> {
    FIRApp.configure();
}, true);
```

But it doesn't and randomly Crashlytics rule signals/mach. This happens due the way it installs handlers:
[FIRCLSContext.m](https://github.com/firebase/firebase-ios-sdk/blob/11.14.0/Crashlytics/Crashlytics/Components/FIRCLSContext.m)  
```objc
FBLPromise* FIRCLSContextInitialize(FIRCLSContextInitData* initData,
                                    FIRCLSFileManager* fileManager) {
   ...
 if (!_firclsContext.readonly->debuggerAttached) {
#if CLS_SIGNAL_SUPPORTED
    dispatch_group_async(group, queue, ^{
      FIRCLSSignalInitialize(&_firclsContext.readonly->signal);
    });
#endif

#if CLS_MACH_EXCEPTION_SUPPORTED
    dispatch_group_async(group, queue, ^{
      FIRCLSMachExceptionInit(&_firclsContext.readonly->machException);
    });
#endif

    dispatch_group_async(group, queue, ^{
      FIRCLSExceptionInitialize(&_firclsContext.readonly->exception,
                                &_firclsContext.writable->exception);
    });
  } else {
    FIRCLSSDKLog("Debugger present - not installing handlers\n");
  }   
}
```

Everything is async, and probably will happen when `Signals.installSignals` exits and will override `RoboVM` handlers...

## Bottom line and fix:
`Crashlytics` is kind of `strange`, have reported few issues there (without any hope):
- [firebase-ios-sdk #15383](https://github.com/firebase/firebase-ios-sdk/issues/15383)
- [firebase-ios-sdk #15384](https://github.com/firebase/firebase-ios-sdk/issues/15384)

Currently, it can be workaround with simple delay:
```java
Signals.installSignals(() -> {
    FIRApp.configure();
    // FIXME: dirty workaround -- wait till Crashlytics install handler async 
    try { Thread.sleep(200); } catch (Exception e) {}
}, true);
```


