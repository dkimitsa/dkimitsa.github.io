---
layout: post
title: 'Kotlin coroutines: Dispatchers.Main context for RoboVM applications'
tags: ['kotlin', 'coroutines']
---
Android has [kotlinx-coroutines-android](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-android) `Dispatcher.Main` that coroutine execution on `main/ui` thread.  
It allows writing an effective suspend logic on main thread as described [on the usage page]():  
```kotlin
fun setup(hello: Text, fab: Circle) {
    GlobalScope.launch(Dispatchers.Main) { // launch coroutine in the main thread
        for (i in 10 downTo 1) { // countdown from 10 to 1 
            hello.text = "Countdown $i ..." // update text
            delay(500) // wait half a second
        }
        hello.text = "Done!"
    }
}
```

# Overview
To make `Dispatchers.Main` available to `kotlinx` following steps to be done:    
<!-- more -->
* [MainDispatcherFactory](https://github.com/dkimitsa/kotlinx.coroutines.robovm/blob/d31121980ce9a9e47400efb2426a1ac2da998d9b/src/main/kotlin/kotlinx/coroutines/robovm/DispatchQueueDispatcher.kt#L58) implementation to be registered through [META-INF/services](https://github.com/dkimitsa/kotlinx.coroutines.robovm/blob/master/src/main/resources/META-INF/services/kotlinx.coroutines.internal.MainDispatcherFactory);
* `Factory` should produce [MainCoroutineDispatcher](https://github.com/dkimitsa/kotlinx.coroutines.robovm/blob/d31121980ce9a9e47400efb2426a1ac2da998d9b/src/main/kotlin/kotlinx/coroutines/robovm/DispatchQueueDispatcher.kt#L27) implementation;
* `MainCoroutineDispatcher` is responsible for dispatching `runnables` on main/ui context. 

# Approaches 
Implementation is a port of [kotlinx-coroutines-android](https://github.com/Kotlin/kotlinx.coroutines/tree/master/ui/kotlinx-coroutines-android) where dispatch was happening using `android.os.Handler`.  
For `RoboVM` there were several options:
- using [Grand Dispatch Central](https://developer.apple.com/documentation/DISPATCH);
- using [Operation Queues](https://developer.apple.com/library/archive/documentation/General/Conceptual/ConcurrencyProgrammingGuide/OperationObjects/OperationObjects.html#//apple_ref/doc/uid/TP40008091-CH101-SW1);
- using [NSObject performSelectorOnMainThread:::](https://developer.apple.com/documentation/objectivec/nsobject/1414900-performselectoronmainthread?language=objc);
- using [NSRunLoop](https://developer.apple.com/documentation/foundation/nsrunloop).

To implement dispatch following was items should be possible in implementation:
- cancellation of pending execution;
- ability to determine if code currently executing inside context;

These are possible with `Grand Dispatch Central` and `Operation Queues`. Last are high level wrapper around `GRC`.

# Implementation
My implementation is based on `GDC` as it is low level and introduces minimal amount of overhead.  
While porting of `Android` part is straightforward cancellation and context identification API was missing in `CocoaTouch`. Following API were added as part of [iOS14.5 bindings](https://github.com/MobiVM/robovm/pull/575/commits/895c74d9fa30f47649a5bed4f856e9905e496aad):
- `dispatch_block_*` for scheduled block cancellation; 
- `dispatch_queue_(get|set)_specific`, `dispatch_get_current_queue` to allow tagging and identification of dispatch_queue as context.

Dispatch with cancellation implemented as bellow:  
```kotlin
    override fun invokeOnTimeout(timeMillis: Long, block: Runnable, context: CoroutineContext): DisposableHandle {
        val dispatchBlock = DispatchBlock.create(DispatchBlockFlags.None, block)
        dispatchQueue.after(timeMillis, TimeUnit.MILLISECONDS, dispatchBlock)
        return object : DisposableHandle {
            override fun dispose() {
                DispatchBlock.cancel(dispatchBlock)
            }
        }
    }
```

And identification:   
```kotlin
    override fun isDispatchNeeded(context: CoroutineContext): Boolean {
        return !invokeImmediately || DispatchQueue.getCurrentSpecific(DispatchQueueAssociatedValues.key) != 
          dispatchQueue.getSpecific(DispatchQueueAssociatedValues.key)
    }

```
# Source code
Source code is available in [dkimitsa/kotlinx.coroutines.robovm](https://github.com/dkimitsa/kotlinx.coroutines.robovm).  
Also it was built and deployed to `sonatype` maven repository and ready for use as dependency:  
```groovy
repositories {
    maven { url 'https://oss.sonatype.org/content/repositories/snapshots' }
}
dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-core:$kotlin_version"
    implementation "io.github.dkimitsa.robovm:kotlinx-coroutines-robovm:0.1-SNAPSHOT"
}
```

# Things to note 
At moment of writing following pull request are not merged and required to be pulled in. And local version of RoboVM to be built from source code.  
* [iOS14.5 bindings](https://github.com/MobiVM/robovm/pull/575);
* if deebugger used: [SMAP parser support for kotline 1.4/1.5](https://github.com/MobiVM/robovm/pull/581).
