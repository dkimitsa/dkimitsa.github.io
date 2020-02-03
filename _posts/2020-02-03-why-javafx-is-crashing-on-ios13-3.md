---
layout: post
title: 'Why JavaFXPorts is crashing on ios13.3'
tags: [bug, ios13, javafx]
---

There was a report on [Gitter channel](https://gitter.im/MobiVM/robovm?at=5e2609e072359f323df05241) saying that there is an issue on ios13.3 in JavaFX port:
> @javasunsFX_twitter
> Anyone has an idea what has changed on iOS 13.3 and JavaFX crashes when calling Stage.show(), which in turn calls “IosWindow -> GlassWindow.m -> _createWindow”?

Issue was confirmed and investigated to check if RoboVM is affected. Crash log looks like bellow:
```
Thread 0 name:  Dispatch queue: com.apple.main-thread
Thread 0 Crashed:
0   libsystem_kernel.dylib        	0x00000001b4f4c930 __abort_with_payload + 8
1   libsystem_kernel.dylib        	0x00000001b4f50f24 abort_with_payload_wrapper_internal + 100
2   libsystem_kernel.dylib        	0x00000001b4f50ec0 abort_with_payload_wrapper_internal + 0
3   libsystem_pthread.dylib       	0x00000001b4e75ed0 DYLD-STUB$$OUTLINED_FUNCTION_0 + 0
4   libsystem_pthread.dylib       	0x00000001b4e750a8 pthread_join + 0
5   Foundation                    	0x00000001b54089d4 +[NSThread currentThread] + 96
6   Watchdog                      	0x00000001049b0858 Java_com_sun_glass_ui_ios_IosWindow__1createWindow + 392
7   Watchdog                      	0x0000000104c69bc0 [J]com.sun.glass.ui.ios.IosWindow._createWindow+ 4856768 (JJI)J + 64

```

And abort payload is `pthread_t was corrupted`. Memory corruption is hard to investigate as cause of it is not same place (and time) where it happened. To investigate issue JavaFX was build as debug and it worked. Future investigation shown that it fails if compiled with `NDEBUG` define. As JavaFX is huge project it is hard to analyze all code.
Quickest way to find a bug in unknown code is to slice it and isolate failing code to few lines.

## Root case
JavaFX source was sliced and ripped to few lines that showed the root case:
- `GLASS_POOL_PUSH` macro causes the issue and corrupts the memory when writing to `_GlassThreadData`;
- `_GlassThreadData` is never allocated because `(GlassThreadData*)pthread_getspecific(GlassThreadDataKey)` returns data;
- return value of `pthread_getspecific` is undefined in case of invalid key (and it is!);
- and key is undefined:

And the root case of issue why key is undefined is the code how its initialized, [GlassApplication.m:635](https://bitbucket.org/javafxports/8u-dev-rt/src/175cb43e5e358015429c965d02491f0705526ba5/modules/graphics/src/main/native-glass/ios/GlassApplication.m#lines-635):
```objc
    assert(pthread_key_create(&GlassThreadDataKey, NULL) == 0);
```

When `NDEBUG` is defined assets are not being compiled and `pthread_key_create(&GlassThreadDataKey, NULL)` is never executed.
It was working on pre ios13.3 just due lucky case as it was corrupting not-critical data.

The fix: is to get `pthread_key_create` out of assert.

## Bottom line
RoboVM compiler is not affected!
