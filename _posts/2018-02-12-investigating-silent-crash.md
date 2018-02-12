---
layout: post
title: 'Tutorial: Investigating silent crash of Release build (e.g. Port death watcher fired)'
date: 2018-02-12 09:00:01 +0200
tags: [tutorial, fix]
---
Release build (shared IPA) crashes when shared with QA team or even being uploaded to Apple Store doesn't give any reason in crash reports. Device console logs shows as much as:
> Feb 12 09:32:12 iPhone assertiond[67] <Notice>: [DemoApp:620] Port death watcher fired.

This can be reproduced with this minimal snippet -- it will produce output while on wire with debugger but once started without it (or from IPA or Apple store) will be silent:
```java
public class Main extends UIApplicationDelegateAdapter {
    @Override
    public boolean didFinishLaunching(UIApplication application, UIApplicationLaunchOptions launchOptions) {
        throw new RuntimeException("You will not see me!");
    }
}
```

With very high chance crash is Java code and it is not causes any log output/crash report because developer didn't configured app so. What has to be done:  
<!-- more -->

## All java exception shall go to NSException
This can be easily enable with one line of code somewhere in `UIApplicationDelegate` implementation:

```java
public class Main extends UIApplicationDelegateAdapter {
    @Override
    public boolean didFinishLaunching(UIApplication application, UIApplicationLaunchOptions launchOptions) {
        ...
        NSException.registerDefaultJavaUncaughtExceptionHandler();
        ...
    }
}
```

This small change give crash report in console to allow work with:
```
Feb 12 10:24:41 iPhone DemoApp(CoreFoundation)[690] <Notice>: *** Terminating app due to uncaught exception 'java.lang.RuntimeException', reason: 'java.lang.RuntimeException: You will not see me!
	at com.mycompany.myapp.Main.didFinishLaunching(Main.java:51)
	at com.mycompany.myapp.Main.$cb$application$didFinishLaunchingWithOptions$(Main.java)
	at org.robovm.apple.uikit.UIApplication.main(Native Method)
	at org.robovm.apple.uikit.UIApplication.main(UIApplication.java:428)
	at com.mycompany.myapp.Main.main(Main.java:64)
```

## Why there is no uncaught Exception console output from JVM itself when running Release code
Even without `NSException.registerDefaultJavaUncaughtExceptionHandler()` there shall be console output about uncaught exception from JVM. First of all lets see how `stdout` and `stderr` find their way to Device console. In release run it got redirected by this code in [UIApplication](https://github.com/MobiVM/robovm/blob/307093d4776c544e84b3e6bd014b703c914332dc/compiler/cocoatouch/src/main/java/org/robovm/apple/uikit/UIApplication.java#L399):
```java
public static <P extends UIApplication, D extends NSObject & UIApplicationDelegate>
    void main(String[] args, Class<P> principalClass, Class<D> delegateClass) {
        ...
        if (System.getenv("ROBOVM_LAUNCH_MODE") == null) {
            if (!(System.err instanceof FoundationLogPrintStream)) {
                System.setErr(new FoundationLogPrintStream());
            }
            if (!(System.out instanceof FoundationLogPrintStream)) {
                System.setOut(new FoundationLogPrintStream());
            }
        }
        ...
}
```

So there is [`FoundationLogPrintStream`](https://github.com/MobiVM/robovm/blob/45b1de02cad5d857918d7604414404323c75f715/compiler/cocoatouch/src/main/java/org/robovm/apple/foundation/FoundationLogPrintStream.java) which will redirects all outputs to `NSLog` but this is happening in following cases:
- explicit `flush` is called;
- as final part of `printLn` methods.

## Still why there is no uncaught Exception console output from JVM even with `FoundationLogPrintStream` included
The reason for this is way how uncaught exception being printed by JVM. It is being done in [ThreadGroup.uncaughtException(Thread t, Throwable e)](https://github.com/MobiVM/robovm/blob/45b1de02cad5d857918d7604414404323c75f715/compiler/rt/libcore/libdvm/src/main/java/java/lang/ThreadGroup.java#L689):
```Java
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else if (Thread.getDefaultUncaughtExceptionHandler() != null) {
        // TODO The spec is unclear regarding this. What do we do?
        Thread.getDefaultUncaughtExceptionHandler().uncaughtException(t, e);
    } else if (!(e instanceof ThreadDeath)) {
        // No parent group, has to be 'system' Thread Group
        e.printStackTrace(System.err);
    }
}
```

Uncaught exception will end up with `e.printStackTrace(System.err)` which seems pretty ok considering that `System.err == FoundationLogPrintStream`, but lets see what is wrong with [Throwable.printStackTrace](https://github.com/MobiVM/robovm/blob/45b1de02cad5d857918d7604414404323c75f715/compiler/rt/libcore/luni/src/main/java/java/lang/Throwable.java#L329). it all ends with lot of code that output information using `err.append()` calls.

Root case of problem is that `FoundationLogPrintStream` is not implementing append() method and all output is that goes after it will not get any `flush()`. This means all buffered data will not go to NSLog() as app will be terminated after uncaught exception.

## quick fix
Using of `NSException.registerDefaultJavaUncaughtExceptionHandler()` would be enough as it does the `flush()` inside;

## long term solution
[PR267](https://github.com/MobiVM/robovm/pull/267). Is to add flush after print of uncaught exception in [ThreadGroup.uncaughtException(Thread t, Throwable e)](https://github.com/MobiVM/robovm/blob/45b1de02cad5d857918d7604414404323c75f715/compiler/rt/libcore/libdvm/src/main/java/java/lang/ThreadGroup.java#L689):
```Java
public void uncaughtException(Thread t, Throwable e) {
    ...
    } else if (!(e instanceof ThreadDeath)) {
        // No parent group, has to be 'system' Thread Group
        e.printStackTrace(System.err);
        System.err.flush();
    }
}
```

## short term solution
It is possible to use own implementation of `PrintStream` which will catch new line symbol and do flush, like this:
```
public class Main extends UIApplicationDelegateAdapter {
    @Override
    public boolean didFinishLaunching(UIApplication application, UIApplicationLaunchOptions launchOptions) {
        ...
        // override robovm bug to allow console logging on release/run
        if (System.getenv("ROBOVM_LAUNCH_MODE") == null) {
            System.setErr(new FoundationLogPrintStream());
            System.setOut(new FoundationLogPrintStream());
        }
        ...
    }

    public class FoundationLogPrintOutputStream extends OutputStream {
    StringBuffer st = new StringBuffer();

    @Override
    public void write(int ch) throws IOException {
        if(ch != 10) {
            this.st.append((char)ch);
        } else {
            try {
                this.flush();
            } catch (IOException ignored) {
            }
        }
    }

    @Override
    public void flush() throws IOException {
        if(this.st.length() > 0) {
            Foundation.log("%@", new NSString(this.st.toString()));
            this.st.setLength(0);
        }
    }
}
```
