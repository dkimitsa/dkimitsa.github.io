---
layout: post
title: 'ObjC interop: state lost - concurrency issues under heavy duty load'
tags: [ObjC, bug]
date: 2025-10-17 00:00:02
---
As continue of [investigating deadlock]({{ site.baseurl }}{% post_url 2025-10-09-objc-investigating-deadlock %}):

`Thread #17` also displays strange stacktrace that normally should not happen:
* native(ObjC code) calls `[myObject description]`;
* tries to marshal pointer(handle) to Java object in `toObjCObject`;
* it doesn't find it in `peer` map and creates new Java instance/wrapper using `ObjCObject.createInstance`.

## What is completely wrong here:

<!-- more -->

* there is valid live pointer at native side of app;
* corresponding live and valid Java object is missing at Java side of app;
* `createInstance()` is called for `MyObject` as it is `@CustomObject` with Java state.

### Why it is wrong:
- `createInstance()` is designed to create Java mirrored instance for `@NativeObject` by just allocating object without calling constructor
- as there is object at native side and no at Java, `createInstance()` will create empty `MyObject` with all state lost.

### How it is supposed to work:
There is special facility in `ObjectOwnershipHelper` that keeps strong references for `@CustomObject` while it's alive at native side.
It overrides `-reatain`/`-release` and only removes it from the map in `-release` when retain count goes to 1.  
As object is alive at native side, this means that its retain count is at least 2:
- at least 1 as it's retained at native side;
- +1 as it there is strong reference for the Java part, at it is also retains ObjC part while Java object is alive.

## Investigation -- logs
While this case reproducible and happens in heavy duty multithread condition (around 500 threads), quickest way is to log and analyze...
Long story short it comes to following sequence of events for object with native/ObjC pointer `0x600000069a10`:

| #    | Timestamp  | Thread | Event          | Java object         |
|------|------------|--------|----------------|---------------------|
| 1    | 870966583  | 90     | alloc          | @4949614496         |
| 2    | 871016000  | 90     | track          | @4949614496         |
| 3    | 871083333  | 90     | retain         | @4949614496, 1 -> 2 |
| 4    | 871208000  | 90     | track          | @4949614496         |                  
| 5    | 871271083  | 90     | retain         | @4949614496, 2 -> 3 |
| 6    | 874986375  | 90     | release        | @4949614496, 3 -> 2 |
| 7    | 2789210000 | 212    | untrack        | @4949614496         |
| 8    | 2789398416 | 212    | release        | @4949614496, 2 -> 1 |
| 9    | 4446006375 | 3      | doDispose      | @4949614496         |
| 10   | 4446007416 | 3      | untrack        | @4949614496         |
| 11   | 4448132166 | 3      | release        | @4949614496, 1 -> 0 |
| 12   | 4449263750 | 3      | dealloc        | @4949614496         |
| 13   | 4449798083 | 255    | alloc          | @5061615856         |
| 14   | 4449924166 | 255    | track          | @5061615856         |
| 15   | 4450324500 | 255    | retain         | @5061615856, 1 -> 2 |
| 16   | 4450996708 | 255    | track          | @5061615856         |
| 17   | 4451265291 | 255    | retain         | @5061615856, 2 -> 3 |
| 18   | 4451824250 | 3      | untrack        | @5061615856         |
| 19   | 5104273541 | 255    | getPeerObject  | NULL                |
| 20   | 5104278612 | 255    | createInstance |                     |

here:
* `Java object` - identifier(address) of Java object, that owns ObjC part;
* alloc -- allocation in NSObject, for the Java object it allocates corresponding ObjC;
* track/untrack -- events in ObjectOwnershipHelper, when it starts keeping/releases Java object counterpart while native part is alive.
* retain/release -- events in ObjectOwnershipHelper, when native/ObjC part is being retained/release by ObjC;
* dealloc -- event when native part is being deallocated (due retain count = 0);
* getPeerObject -- returns known Java object for native pointer.
* createInstance -- unexpected creation of `@CustomObject`

## The flow, explained
### #1: Object is allocated at Java side
for the Java object `@4949614496`, created native object `0x600000069a10`, retain count = 1.  
References (at Java side):
- as weak reference: ObjCObject.peers[0x600000069a10] = javaObjc@4949614496
  It is not being kept in `ObjectOwnershipHelper.CUSTOM_OBJECTS` as it is not being kept yet at native side.

### #2 as result of #3
Java object is being marshaled to native side as `autoreleased`.
And `retain` is being called by auto-release pool, increasing retain count 1 -> 2.
Before calling `#2 retain`, ObjectOwnershipHelper makes a strong reference to it #3.  
References (at Java side):
- as weak: `ObjCObject.peers[0x600000069a10] = javaObjc@4949614496`
- as strong: `ObjectOwnershipHelper.CUSTOM_OBJECTS[0x600000069a10] = javaObjc@4949614496`

native side:
- in auto-release pool.
- retainCount = 2

### #4, #5
Object used in Swift code: once received it retains it (retain count 2 -> 3) and once object is out of usage scope release (Retain count 3 -> 2).  
At this moment native part doesn't own it anymore.
As retain count doesn't go to bottom threshold (<= 1) no changes in `ObjectOwnershipHelper` happens.
References (at Java side):
- as weak: `ObjCObject.peers[0x600000069a10] = javaObjc@4949614496`
- as strong: `ObjectOwnershipHelper.CUSTOM_OBJECTS[0x600000069a10] = javaObjc@4949614496`

native side:
- in auto-release pool.
- retainCount = 2

### #7, #8
While #1-#6 was happening in same thread (90), this release #8 is happening in thread 212. This is autorelease flush task/thread, and it flushes Pools after they are not used.  
As in retain count goes 2 -> 1 this means that native part is not kept in native part anymore but being kept by Java object itself only.
So `ObjectOwnershipHelper` untrack the pointer 0x600000069a10 and stops keeping strong reference to javaObjc@4949614496.
References (at Java side):
- as weak: `ObjCObject.peers[0x600000069a10] = javaObjc@4949614496`
  
Native side:
- in auto-release pool.
- retainCount = 1 (javaObjc@4949614496 keeps it )

### #9, #10, #11, #12
As Java object is not hard-reference anywhere anymore, `javaObjc@4949614496` is being garbage collected.  
Thread 3 is finalizer thread that call garbage collected objects `finalize()`.  
Here followings happens:
- `ObjCObject` removes `0x600000069a10` from `ObjCObject.peers`
- `#9 NSObject.doDispose` calls `#11 release()` as dead Java object is not responsible for native object anymore;
- due `#11`, ObjectOwnershipHelper tries to remove(`#10`) any object for `0x600000069a10` pointer. At this moment it was already removed by #7.
- as retains count is 0, native release calls `#12 dealloc`.

References (at Java side):
- none

native side:
- none

IMPORTANT NOTE: after dealloc native memory is available for new allocation.

### #13, #14, #15, #16
At same time, in parallel ~~universe~~ Thread #255 create new instance of `javaObjc@5061615856` and it for native part new memory is allocated, as previous used memory was released in #12 we receive same pointer `0x600000069a10`.
Then it is being used somewhere in native code so got retained `#15`, `17` and being strongly referenced by `ObjectOwnershipHelper` (`#14`, `#16`).  
References (at Java side):
- as weak: `ObjCObject.peers[0x600000069a10] = javaObjc@5061615856`
- as strong: `ObjectOwnershipHelper.CUSTOM_OBJECTS[0x600000069a10] = javaObjc@5061615856`

native side:
- in auto-release pool, in swift usage scope.
- retainCount = 3


### #18
in parallel ~~universe~~ thread 3, just finishing `dealloc`, removes strong reference for `javaObjc@4949614496`, as
`0x600000069a10` pointer is not associated with `javaObjc@4949614496` anymore(as last was garbage collected).

HERE IS BUG #1:
- `ObjectOwnershipHelper.CUSTOM_OBJECTS[0x600000069a10]` keeps now `javaObjc@5061615856`, not `javaObjc@4949614496`
  As result `javaObjc@5061615856` is not referenced anymore and is subject to be garbage collected.    
  Thus, there will be native object and no corresponding Java one.

References (at Java side):
- as weak: `ObjCObject.peers[0x600000069a10] = javaObjc@5061615856`

native side:
- in auto-release pool, in swift usage scope.
- retainCount = 3

HISTORICAL NOTE:
Why this is all happening in de-alloc anyway, as retain/release count is enough to make decision.
1. at moment of dealloc there should be no references to this pointer in both `ObjCObject.peers` and/or `ObjectOwnershipHelper.CUSTOM_OBJECTS`.
2. but due bugs in native part, it could be over released (more release called than retain). In this case `ObjCObject.peers` might still contain pointer.
3. reference is removed after and not before `dealloc` as destructor native code might call retain/release while being inside `dealloc`. All this will produce `ObjectOwnershipHelper` to make another references.

### #19, #20
Ok, due bug `#18` had removed reference to `javaObjc@5061615856` by `0x600000069a10` there was still valid reference in `ObjCObject.peers` after #18.  
`getPeerObject()` returned `null`, code for it is straight forward:
```java
static class ObjCObjectRef extends WeakReference<ObjCObject> {}
private static final LongMap<ObjCObjectRef> peers = new LongMap<>();
protected static <T extends ObjCObject> T getPeerObject(long handle) {
    synchronized (objcBridgeLock) {
        ObjCObjectRef ref = peers.get(handle);
        T o = ref != null ? (T) ref.get() : null;
        return o;
    }
}
```

HERE IS BUG #2: `ObjCObject.peers` might lose references to `@CustomObjects` that are being kept in `ObjectOwnershipHelper` or any other Java code.

# Fixes
* BUG1: `dealoc` + un-reference shall be done atomic, e.g. protected by same ObjC-lock;
* BUG2: `getPeerObject` has to be modified to look inside `ObjectOwnershipHelper.CUSTOM_OBJECTS` if nothing found in `peers`

Fix proposed as [PR#821](https://github.com/MobiVM/robovm/pull/821).
