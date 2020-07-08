---
layout: post
title: 'Crashlytics: proper java stacktrace'
tags: [hacking, fix]
---
This post continues tutorial for [proper initialization of crash reporters t]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %}).  
Option to track java exception was to use `NSException.registerDefaultJavaUncaughtExceptionHandler()`. It builds string presentation of exception creates NSException with this text as reason. At same time stack traces will be messed and will point to code where NSException is created.  

## FIRExceptionModel
Firebase Crashlytics provide special API `FIRExceptionModel`:
> The Firebase Crashlytics Exception Model provides a way to report custom exceptions to Crashlytics that came from a runtime environment outside of the native platform Crashlytics is running in.

Exact case for RoboVM. To have it working its enough to setup `Thread.setDefaultUncaughtExceptionHandler` as bellow to convert exception stack traces into `FIRExceptionModel` elements:  
```java
Thread.setDefaultUncaughtExceptionHandler((thread, ex) -> {
    FIRExceptionModel model = new FIRExceptionModel(ex.getClass().getName(), ex.getMessage() != null ? ex.getMessage() : "");
    NSMutableArray<FIRStackFrame> modelFrames = new NSMutableArray<>();
    StackTraceElement[] stackTraces = ex.getStackTrace();
    if (stackTraces != null) {
        for (StackTraceElement st : stackTraces) {
            String symbol = st.getClassName() + '.' + st.getMethodName();
            String fileName;
            int lineNo = st.getLineNumber();
            if (lineNo < 0)
                lineNo = 0;
            if (st.isNativeMethod()) {
                fileName = "Native Method";
            } else {
                fileName = st.getFileName();
                if (fileName == null)
                    fileName = "Unknown Source";
            }

            FIRStackFrame frame = new FIRStackFrame(symbol, fileName, lineNo);
            modelFrames.add(frame);
        }
    }
    model.setStackTrace(modelFrames);
    FIRCrashlytics.crashlytics().recordExceptionModel(model);

    // kill app
    throw new ThreadDeath();
});
```
