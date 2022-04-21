---
layout: post
title: 'Fix for nasty `rvmCreateVM SIGABRT` crash'
tags: [fix, debugger, vm]
---
This issue was mentioned multiple times over all channels and all reports have the following in common:
- issue usually was seen in crash reporters;
- no steps or information how to reproduce; 
- no simple project to reproduce;
- production source code can't be shared for investigation;
- crash is related to heavy network activities;
- crash log looks similar to bellow:

```
rvmCreateVM
SIGABRT ABORT 0x00000001d5219414

Crashed: Thread
0  libsystem_kernel.dylib         0x1d5219414 __pthread_kill + 8
1  libsystem_pthread.dylib        0x1f2d74b50 pthread_kill + 272
2  libsystem_c.dylib              0x1b06f7b74 abort + 104
3  RoboVMMobile                   0x1060a77b0 rvmCreateVM + 4386699184
4  RoboVMMobile                   0x1060a51ec rvmInitialize + 4386689516
5  RoboVMMobile                   0x10609d9a8 _bcInitializeClass + 4386658728
```

Things changed once `Benjamin Schulte @MrStahlfelge` [reported](https://gitter.im/MobiVM/robovm?at=624364636b91242320598a9c) unknown rare crash that was happening approximately once per 100 launches and [ErgoWallet](https://github.com/ergoplatform/ergo-wallet-app) is opensources.
1 crash per 100 launches sound good to track the bug. 

# Root case
<!-- more -->
Issue was reproduced and stack trace for debug code looks more informative than production one (there is an issue[https://github.com/MobiVM/robovm/issues/522] for this):
> Assertion failed: (CLASS_IS_STATE_LOADED(clazz)), function rvmInitialize, file class.c, line 1173.

And its actual state is `CLASS_STATE_ALLOCATED` which looks odd. Considering that `bcInitializeClass` logical steps looks as following:
```
bcInitializeClass
  ldcClass
     rvmFindClassUsingLoader
         java.lang.ClassLoader.loadClass()
              rvmFindLoadedClass
                 getLoadedClass (from hash)
              rvmFindClassInClasspathForLoader
                 synchronized {   
                    loadUserClass-loadClass-createClass()
                       rvmAllocateClass -> CLASS_STATE_ALLOCATED
                       rvmRegisterClass()
                          addLoadedClass (to hash map)
                          -> CLASS_STATE_LOADED
                 }
  rvmInitialize
     synchronized {
        // expects CLASS_STATE_LOADED
        // no ineresting as crash is happening here
     } 
```

Call flow looks straight forward and from logical flow it not expected to have `CLASS_STATE_ALLOCATED` in `rvmInitialize`.  
Heavy load is concurrency sensitive and classic synchronization problem is here.  
The issue is triggered by concurrent loading of same class from multiple threads.  
Scenario is following:  

### Thread1:
1. didn't find loaded class;
2. goes creating one in `createClass()`;
3. move class state to `CLASS_STATE_ALLOCATED`;
4. and register class into loaded classes hashmap by `addLoadedClass`;
5. moves class to `CLASS_STATE_LOADED`;
6. goes to `rvmInitialize()`

### Thread2 and trouble:
- tries to `rvmFindLoadedClass` while Thread1 somewhere between 4. and 5. 
- as class in hash map it picks it from there and takes a shortcuts directly to `rvmInitialize()`.
- Assertion failed: (CLASS_IS_STATE_LOADED(clazz))

### Problems is here: 
- `rvmFindLoadedClass` allows not synchronized access to loaded class hash map. As result class in middle of initialization is returned as loaded (this actually a root case);
- `rvmFindLoadedClass` should not return classes in state other than `CLASS_STATE_INITIALIZED` otherwise `java.lang.ClassLoader.loadClass()` might return not initialized classes.

# The fix
```patch
 Class* rvmFindLoadedClass(Env* env, const char* className, Object* classLoader) {
+    obtainClassLock();
     Class* clazz = getLoadedClass(env, className);
+    releaseClassLock();
     if (rvmExceptionOccurred(env)) return NULL;
+    if (clazz && !CLASS_IS_STATE_INITIALIZED(clazz)) return NULL;
     return clazz;
 }
```

# Code
The fix was proposed as [PR648](https://github.com/MobiVM/robovm/pull/648)

# Tips how to catch such kinds of bugs
## Using debug version of RoboVM
Native code has to be build in debug mode:
> compiler/vm/build.sh --build=debug

To use debug libraries RoboVM to be run in debug mode by specifying `-DROBOVM_DEV_ROOT={path to compiler dir}` dev root home;

## Ability to reproducing 
Crash rate 1 from 100 might be too boring for manual reproduction (which might not happen in debug mode). To get proof is useful to run an automated batch to see if it reproducible. 
Following run and restart app in simulator multiple times:
> for i in {1..500}; do echo "-"; echo "-"; echo $i ; timeout 4 xcrun simctl launch --console 7F276E93-4A20-49F8-944B-227386A0FA80 org.ergoplatform.ios ; done

NP: `timeout` is part of `coreutils`, so `brew install coreutils`.

## Logging
It's not 100% reproducible, good approach is to put `printf`/asserts in lines of interest and launch batch to get the crash. And analyze the output. Then put same in better places :) 

## Read code
When possible problem and code area is localised it might be enough to read the code. 

## Crash log symbolication
Console often fails to symbolicate crash as it messes with DSYMs in indexed location. It's quicker to symbolicate crashes manually. 
But apple moved to `.ips` json-like format of crash logs and sadly `symbolicatecrash` script is not able to handle it.
Quickest way is to use `atos`:
Open `.ips` crash report in `Console.app`, find stack trace of interest:
```
Thread 20 Crashed:
0   libsystem_kernel.dylib        	       0x1cba39e60 __pthread_kill + 8
1   libsystem_pthread.dylib       	       0x1cba8d3c0 pthread_kill + 256
2   libsystem_c.dylib             	       0x1801023f4 abort + 124
3   libsystem_c.dylib             	       0x180101910 __assert_rtn + 268
4   ErgoWallet                    	       0x103dba5c8 0x100858000 + 55977416
5   ErgoWallet                    	       0x103dac4ac 0x100858000 + 55919788
6   ErgoWallet                    	       0x103dac41c 0x100858000 + 55919644
7   ErgoWallet                    	       0x102172554 0x100858000 + 26322260
8   ErgoWallet                    	       0x10217245c 0x100858000 + 26322012
9   ErgoWallet                    	       0x1014bd54c 0x100858000 + 12997964
10  ErgoWallet                    	       0x10215af84 0x100858000 + 26226564
```

Here `0x100858000` in second column is the image load address, and first column is a stacktrace addresses.
Switch to `robovm-build/tmp/Unnamed/ios/arm64-simulator` where `ErgoWallet.app.dSYM` is located and run:
> atos -o ErgoWallet.app.dSYM/Contents/Resources/DWARF/ErgoWallet -l {load_address} {stacktrace1} {stacktrace2} ...

e.g.
> atos -o ErgoWallet.app.dSYM/Contents/Resources/DWARF/ErgoWallet -l 0x100858000 0x103dba5c8 0x103dac4ac 0x103dac41c 0x102172554

## Using Xcode
It will catch `abort` and will allow to evaluate stacktraces/thread/data content. Very useful.

### Option one: attach to running process
- Open/Create any dummy iOS project in Xcode;
- Make sure robovm app is running in simulator;
- Check that exactly same simulator is selected in Xcode;
- use menu "Debug/Attach to Process by PID or Name";
- enter image name, e.g. `ErgoWallet`. And attach;
- wait for crash, set symbolic breakpoints whatever.

Doesn't work when app crashes on startup. 

### Option two: make xcode to run pre-build application
- build and run robovm app on simulator (e.g. binary to be build);
- open Product->Scheme->Edit scheme... dialog;
- in info tab, click `Executable`, select `Other`;
- in directory picked dialog navigate to `robovm-build/tmp/Unnamed/ios/arm64-simulator` and select application, e.g. `ErgoWallet.app`.

Now when `Run` is hit, Xcode will build dummy project but deploy and launch robovm one. 

Happy coding !

