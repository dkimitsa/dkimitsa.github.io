---
layout: post
title: 'bugfix #357, #387: CGBitmapContext.create crashes'
tags: [fix, debugger, eclipse]
---
**History** [CGBitmapContext.create fails on real device for big sizes #387](https://github.com/MobiVM/robovm/issues/387)  
**Fix** [PR394](https://github.com/MobiVM/robovm/pull/394)  

## Root case
`CGBitmapContext.create(byte[] ... )` method implemented as bellow:
```java
BytePtr ptr = new BytePtr();
ptr.set(data);
return create(ptr.as(IntPtr.class), width, height, bitsPerComponent, bytesPerRow, space, bitmapInfo, releaseCallback);
```
interested moments here:
- `new BytePtr();` allocate struct that contains one byte (size of struct is one byte);
-  `ptr.set(data)` just mem copies data over this one byte struct and corrupts all memory after it;


## The fix
<!-- more -->
*First* of all is to get rid of all pointers data types like (`IntPtr`) and introduce API that accepts array data only.  

*Second* is to make sure buffer allocated for data is not GC-ed while instance of `CGBitmapContext` exist, e.g. code like this will cause buffer to be GC-ed one moment and this will cause memory corruption on operation with `CGBitmapContext`:
```java
CGBitmapContext.create(new byte[128*128*4], ...)
``` 

This is workarounded by keeping a reference to data and null-in it when it is not required anymore by CGBitmapContext.  
To achieve this code was addapted to utilize `CGBitmapContextCreateWithData` with release callback for every `CGBitmapContext.create()`, and keep pointer to data in corresponding callback object:
```java
private static class DataHolderReleaseCallback implements ReleaseDataCallback {
    private Object data;
    public DataHolderReleaseCallback(Object data) {this.data = data;}
    @Override
    public void release(long ptr) {this.data = null;}
}
```

*Third* is was required to rework the map of callbacks as previously callbacks were added there but never removed. This caused memory leaks as objects got stuck there forewer. Solution is to remove object on callback:
```java
private static void cbReleaseData(@Pointer long refcon, @Pointer long data) {
    ReleaseDataCallback callback;
    synchronized (callbacks) {
        callback = callbacks.remove(refcon);
    }
    if (callback != null)
        callback.release(data);
}
```

And to remove on dispose (in case callback is not called):
```java
@Override
protected void dispose(boolean finalizing) {
    super.dispose(finalizing);

    // remove registered callback otherwise it might stuck in callbacks
    if (releaseInfo > 0) {
        synchronized (callbacks) {
            callbacks.remove(releaseInfo);
            releaseInfo = 0;
        }
    }
}
```
