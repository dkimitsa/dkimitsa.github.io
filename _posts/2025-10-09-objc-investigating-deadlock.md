---
layout: post
title: 'ObjC interop: deadlock — concurrency issues under heavy load'
tags: [objc, tutorial, bug]
date: 2025-10-17 00:00:01
---

Due to a deadlock, a synchronization object in ObjC-related code became unavailable, causing the application to hang while trying to acquire a lock on it. It could also crash if the garbage collector (GC) was involved at that moment:
```
[ERROR] android.System: com.example.TestObject.finalize() timed out after 10 seconds
[ERROR] android.System: java.util.concurrent.TimeoutException: com.example.TestObject.finalize() timed out after 10 seconds
    at org.robovm.objc.ObjCObject$ObjectOwnershipHelper.release(ObjCObject.java:525)
    at org.robovm.objc.ObjCRuntime.void_objc_msgSend(Native Method)
    at org.robovm.apple.foundation.NSObject.release(NSObject.java:228)
    at org.robovm.apple.foundation.NSObject.doDispose(NSObject.java:212)
    at org.robovm.objc.ObjCObject.dispose(ObjCObject.java:148)
    at org.robovm.objc.ObjCObject.finalize(ObjCObject.java:135)
    at java.lang.Daemons$FinalizerDaemon.doFinalize(Daemons.java:172)
    at java.lang.Daemons$FinalizerDaemon.run(Daemons.java:155)
    at java.lang.Thread.run(Thread.java:869)
```

## Getting the stack traces

<!-- more -->

Due to project-specific details (RoboVM is used as a framework), the simplest way to analyze the issue is to attach the Xcode debugger to the process once it hangs but before it crashes:

1. Create and use any dummy iOS Xcode project.
2. Select the expected simulator (where the application is installed).
3. In the menu, go to **Debug → Attach to Process by PID or Name...**, and specify the binary name (e.g., `Main`, `IOSLauncher`) — but do **not** hit *Attach* yet.
4. Hit *Attach* once the process hangs.

Then, find any thread locked in `__psynch_mutexwait`, which was invoked from `ObjCObject`, for example:
```
Thread 9 Queue : com.apple.root.default-qos (concurrent)
#0  0x00000001044fb564 in __psynch_mutexwait ()
#1  0x0000000103a67b10 in _pthread_mutex_firstfit_lock_wait ()
#2  0x0000000103a656b4 in _pthread_mutex_firstfit_lock_slow ()
#3  0x0000000121929674 in lockMonitor ()
#4  0x0000000121929498 in rvmLockObject ()
#5  0x0000000121916868 in _bcMonitorEnter ()
#6  0x000000012157d664 in [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retainObject(J)V at ObjCObject$ObjectOwnershipHelper.java:498
```

This stack trace corresponds to the following code:
```java
public static void retainObject(long self) {
    synchronized (CUSTOM_OBJECTS) {  //  <<< WAITING HERE
        ObjCObject obj = ObjCObject.getPeerObject(self);
        CUSTOM_OBJECTS.put(self, obj);
    }
}
```

Since this is a heavy-load case with around 500 threads, many of them might also be locked or deadlocked on the same semaphore.  
One option is to inspect all threads and check for possible deadlocks — but that’s tedious. A better way is to look into the `pthread_mutex` internals.

## Finding the locked thread by inspecting `pthread_mutex`

In Xcode LLDB, the `pthread_mutex` passed to `__psynch_mutexwait` looks like this:
```
(lldb) p/x *(pthread_mutex_t*)$x0
(pthread_mutex_t) (__sig = 0x000000004d55545a, __opaque = "....")
```

This corresponds to the declaration in `_pthread_types.h`:
```c
struct _opaque_pthread_mutex_t {
    long __sig;
    char __opaque[__PTHREAD_MUTEX_SIZE__];
};
```

All internals are hidden inside `__opaque`, but its structure can be found in [apple-oss-distributions/libpthread/types_internal.h](https://github.com/apple-oss-distributions/libpthread/blob/1ebf56b3a702df53213c2996e5e128a535d2577e/src/types_internal.h#L219):
```c
struct pthread_mutex_s {
    long sig;
    _pthread_lock lock;
    union {
        uint32_t value;
        struct pthread_mutex_options_s options;
    } mtxopts;
    int16_t prioceiling;
    int16_t priority;
#if defined(__LP64__)
    uint32_t _pad;
#endif
    union {
        struct {
            uint32_t m_tid[2]; // thread id of thread that has mutex locked
            uint32_t m_seq[2]; // mutex sequence id
            uint32_t m_mis[2]; // for misaligned locks m_tid/m_seq will span into here
        } psynch;
        struct _pthread_mutex_ulock_s ulock;
    };
#if defined(__LP64__)
    uint32_t _reserved[4];
#else
    uint32_t _reserved[1];
#endif
};
```

The field `uint32_t m_tid[2]` contains the thread ID of the thread that currently holds the mutex.  
Reading memory under this structure gives the following:
```
(lldb) memory read --size 8 --format x --count 8 $x0
0x11a49e4f8: 0x000000004d555458 0x000120a800000000
0x11a49e508: 0x4d55545800000000 0x0000000001c7afc3
0x11a49e518: 0x0031420000315602 0xffffffffffffffff
0x11a49e528: 0xfffffffee5b61b07 0x4d5554584d555458
```

`0x1c7afc3` looks like a thread ID — let’s check it in the thread list:
```
(lldb) thread list
Process 4828 stopped
...
  thread #9: tid = 0x1c7aedc, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
...
  thread #16: tid = 0x1c7aee4, 0x00000001044f8b70 libsystem_kernel.dylib`mach_msg2_trap + 8, name = 'com.apple.uikit.eventfetch-thread'
  thread #17: tid = 0x1c7aee5, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
  thread #18: tid = 0x1c7aee6, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
...  
  thread #199: tid = 0x1c7afc2, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
  thread #200: tid = 0x1c7afc3, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
  thread #201: tid = 0x1c7afc4, 0x00000001044fb564 libsystem_kernel.dylib`__psynch_mutexwait + 8, queue = 'com.apple.root.default-qos'
```

And here it is: `thread #200: tid = 0x1c7afc3`.  
CONCLUSION: `Thread #9` is waiting to lock on `CUSTOM_OBJECTS`, that is owned by `Thread #200`.  

Checking stacktrace of `thread #200`:
```
Thread 200 Queue : com.apple.root.default-qos (concurrent)
#0	0x00000001044fb564 in __psynch_mutexwait ()
#1	0x0000000103a67b10 in _pthread_mutex_firstfit_lock_wait ()
#2	0x0000000103a656b4 in _pthread_mutex_firstfit_lock_slow ()
#3	0x0000000121929674 in lockMonitor ()
#4	0x0000000121929498 in rvmLockObject ()
#5	0x0000000121916868 in _bcMonitorEnter ()
#6	0x0000000121579b70 in [J]org.robovm.objc.ObjCObject.getPeerObject(J)Lorg/robovm/objc/ObjCObject; at ObjCObject.java:182
#7	0x00000001215791d4 in [j]org.robovm.objc.ObjCObject.getPeerObject(J)Lorg/robovm/objc/ObjCObject;[clinit] ()
#8	0x000000012157d678 in [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retainObject(J)V 
#9	0x000000012157d7a4 in [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J
#10	0x000000012157cd08 in [j]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J[callback] ()
#11	0x000000010fdb2a0c in Kotlin_ObjCExport_convertUnmappedObjCObject ()
```

and it is waiting for following lock:
```java
protected static <T extends ObjCObject> T getPeerObject(long handle) {
    synchronized (objcBridgeLock) { // <<< WAITING HERE
        ObjCObjectRef ref = peers.get(handle);
        T o = ref != null ? (T) ref.get() : null;
        return o;
    }
}
```

using approach with `pthread_mutex` checking who is owning `objcBridgeLock`:
```
(lldb) memory read --size 8 --format x --count 8 $x0
0x11ad42f08: 0x000000004d555458 0x000120a800000000
0x11ad42f18: 0x4d55545800000000 0x0000000001c7aee5
0x11ad42f28: 0x008385000083a302 0xffffffffffffffff
0x11ad42f38: 0xfffffffee52bd0f7 0x4d5554584d555458
```

`0x1c7aee5` corresponds to `thread #17`.  
**Conclusion:** `Thread #200` owns `CUSTOM_OBJECTS` and waits on `objcBridgeLock`, owned by `Thread #17`.

Checking `Thread #17` stacktrace: 
```
Thread 17 Queue : com.apple.root.default-qos (concurrent)
#0	0x00000001044fb564 in __psynch_mutexwait ()
#1	0x0000000103a67b10 in _pthread_mutex_firstfit_lock_wait ()
#2	0x0000000103a656b4 in _pthread_mutex_firstfit_lock_slow ()
#3	0x0000000121929674 in lockMonitor ()
#4	0x0000000121929498 in rvmLockObject ()
#5	0x0000000121916868 in _bcMonitorEnter ()
#6	0x000000012157d664 in [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retainObject(J)V ObjCObject$ObjectOwnershipHelper.java:498
#7	0x000000012157d7a4 in [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J 
#8	0x000000012157cd08 in [j]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J[callback] ()
#9	0x00000001215839e8 in [J]org.robovm.objc.ObjCRuntime.void_objc_msgSend(JJ)V ()
#10	0x000000012157f5a0 in [j]org.robovm.objc.ObjCRuntime.void_objc_msgSend(JJ)V[clinit] ()
#11	0x0000000120fd295c in [J]org.robovm.apple.foundation.NSObject.retain(J)V
#12	0x0000000120fd26a4 in [J]org.robovm.apple.foundation.NSObject.afterMarshaled(I)V
#13	0x000000012157aaf4 in [J]org.robovm.objc.ObjCObject.createInstance(Lorg/robovm/objc/ObjCClass;JIZ)Lorg/robovm/objc/ObjCObject;
#14	0x000000012157a67c in [J]org.robovm.objc.ObjCObject.toObjCObject(Ljava/lang/Class;JIZ)Lorg/robovm/objc/ObjCObject; 
#15	0x000000012157a2f4 in [J]org.robovm.objc.ObjCObject.toObjCObject(Ljava/lang/Class;JI)Lorg/robovm/objc/ObjCObject; 
#16	0x0000000121579280 in [j]org.robovm.objc.ObjCObject.toObjCObject(Ljava/lang/Class;JI)Lorg/robovm/objc/ObjCObject;[clinit] ()
#17	0x0000000120fda3e8 in [J]org.robovm.apple.foundation.NSObject$Marshaler.toObject(Ljava/lang/Class;JJZ)Lorg/robovm/apple/foundation/NSObject;
#18	0x0000000120fda3bc in [J]org.robovm.apple.foundation.NSObject$Marshaler.toObject(Ljava/lang/Class;JJ)Lorg/robovm/apple/foundation/NSObject;
#19	0x0000000120fda2a8 in [j]org.robovm.apple.foundation.NSObject$Marshaler.toObject(Ljava/lang/Class;JJ)Lorg/robovm/apple/foundation/NSObject;[clinit] ()
#20	0x000000011ecc9810 in [j]com.example.MyObject.$cb$description(Lcom/coinomi/ios/sdk/com.example.MyObject;Lorg/robovm/objc/Selector;)Ljava/lang/String;[callback] ()
```

It waits for `CUSTOM_OBJECTS` in `retainObject` same as `Thread 9`.   
From `Thread 9` is already known that `Thread 200` owns `CUSTOM_OBJECTS`, but let`s check:
```
(lldb) memory read --size 8 --format x --count 8 $x0
0x11a49e4f8: 0x000000004d555458 0x000120a800000000
0x11a49e508: 0x4d55545800000000 0x0000000001c7afc3
0x11a49e518: 0x0031420000315602 0xffffffffffffffff
0x11a49e528: 0xfffffffee5b61b07 0x4d5554584d555458
```

Thread Id `0x1c7afc3` corresponds to `Thread 200`.  
**Conclusion:** `Thread #17` is waiting to lock on `CUSTOM_OBJECTS`, owned by `Thread #200`.
 
Getting all conclusions together:
* `Thread #9` is waiting on `CUSTOM_OBJECTS`, owned by `Thread #200`.  
* `Thread #200` owns `CUSTOM_OBJECTS` and is waiting on `objcBridgeLock`, owned by `Thread #17`.
* `Thread #17` owns `objcBridgeLock` and is waiting on `CUSTOM_OBJECTS`, owned by `Thread #200`.

And there it is — a **classic deadlock** involving two locks and two threads:

`Thread #200`:
```java
public static void retainObject(long self) {
    synchronized (CUSTOM_OBJECTS) {     // owns CUSTOM_OBJECTS
        ObjCObject.getPeerObject(self); // waits for objcBridgeLock
    }
}
```

`Thread #17`:
```java
public static <T extends ObjCObject> T toObjCObject(Class<T> cls, long handle, ...) {
    synchronized (objcBridgeLock) {   // owns objcBridgeLock
        // ...
        NSObject.retain() {
            ObjectOwnershipHelper.retainObject(self) {
                synchronized (CUSTOM_OBJECTS) { // waits for CUSTOM_OBJECTS
                    ObjCObject.getPeerObject(self);
                }
            }
        }
    }
}
```

## The fix
Single `objcBridgeLock` will be used for synchronization instead of `CUSTOM_OBJECTS`.

Fix proposed as [PR#821](https://github.com/MobiVM/robovm/pull/821).
