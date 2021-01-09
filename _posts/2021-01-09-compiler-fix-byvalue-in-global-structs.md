---
layout: post
title: 'Compiler: fix for @ByValue fields got returned as @ByRef'
tags: ['fix']
---
@vaaneeunbnd reported an issue at [gitter](https://gitter.im/MobiVM/robovm?at=5ff3acf74eba353cdf1aabff):  
> I am using GLKMatrix4 as a modelviewmatrix in my robovm app but whenever I try to apply translation to it:
> `modelViewMatrix.createTranslation(0.0f, 0.0f, -1.0f);`
> It crashes without a debug log, can someone please help me with this

But diagnostic report contains a crash log:  
```
Exception Type:        EXC_BAD_ACCESS (SIGBUS)
Exception Codes:       KERN_PROTECTION_FAILURE at 0x0000000106a416e0
Exception Note:        EXC_CORPSE_NOTIFY
VM Regions Near 0x106a416e0:
--> __TEXT                      106a18000-106a48000 
0   Java_libcore_io_Memory_pokeInt + 11
1   [J]libcore.io.Memory.pokeInt(JIZ)V + 119
2   [j]libcore.io.Memory.pokeInt(JIZ)V[clinit] + 71
3   [j]libcore.io.Memory.pokeInt(JIZ)V[Invokestatic(java/nio/MemoryBlock)] + 14
4   [J]java.nio.MemoryBlock.pokeInt(IILjava/nio/ByteOrder;)V + 184 (MemoryBlock.java:220)
5   [j]java.nio.MemoryBlock.pokeInt(IILjava/nio/ByteOrder;)V[Invokevirtual(java/nio/Dire
6   [J]java.nio.DirectByteBuffer.putFloat(IF)Ljava/nio/ByteBuffer; + 393 (DirectByteBuffer.java
7   [j]java.nio.ByteBuffer.putFloat(IF)Ljava/nio/ByteBuffer;[Invokevirtual(java/nio/Byte
8   [J]java.nio.ByteBufferAsFloatBuffer.put(IF)Ljava/nio/FloatBuffer; + 208 (Byt
9   [j]java.nio.FloatBuffer.put(IF)Ljava/nio/FloatBuffer;[Invokevirtual(org/robovm/apple/glki
10  [J]org.robovm.apple.glkit.GLKMatrix4.createTranslation(FFF)Lorg/robovm/apple/glkit/GLKM
11  [j]org.robovm.apple.glkit.GLKMatrix4.createTranslation(FFF)Lorg/robovm/apple/glkit/GLKM
12  [j]org.robovm.apple.glkit.GLKMatrix4.createTranslation(FFF)Lorg/robovm/apple/glkit/GLKMatrix4;[Invokestatic(com/mycompany/myapp/MyViewController)] + 9
```

And code that triggered it looks like the bellow:  
<!-- more -->
```java
public static GLKMatrix4 createTranslation(float tx, float ty, float tz) {
    GLKMatrix4 m = Identity();
    FloatBuffer m_m = m.getM();
    m_m.put(12, tx);
    m_m.put(13, ty);
    m_m.put(14, tz);
    return m;
}
```

`m_m.put(12, tx)` causes `KERN_PROTECTION_FAILURE` due writing to prohibited memory region. `GLKMatrix4` was returned by `Identity()` seems valid but can't be modified. It obtains value from following global variable:
```java
@GlobalValue(symbol="GLKMatrix4Identity", optional=true)
public static native @ByVal GLKMatrix4 Identity();    
```

Quick check shows that its is located not in `__DATA` segment and linker might put it to `__TEXT` where any modifications are not possible:  
```
nm GLKit | grep GLKMatrix4Identity
00000000000296b0 S _GLKMatrix4Identity
```

But `Identity()` is defined with `@ByVal` annotation that means it should return it by-value, e.g. make a copy of data on the heap. 

# Root case -- `@ByVal` doesn't affect getters for GlobalValues/Structs
Existing implementation just marshal the pointer from GlobalValue/Struct member that corresponds to the `@ByRef` behaviour. 
Here is an example that demonstrates the bug:  
```java
CGRect rect = new CGRect(1, 2, 3, 4);
CGPoint origin = rect.getOrigin();
System.out.println("x=" + origin.getX() +", y=" + origin.getY());
rect.setOrigin(new CGPoint(5, 6));
System.out.println("x=" + origin.getX() +", y=" + origin.getY());
```  
prints:  
```
x=1.0, y=2.0
x=5.0, y=6.0
```

The fix is to make a copy of structure onto the heap and then marshal the pointer. 
Fix is delivered as [PR550](https://github.com/MobiVM/robovm/pull/550). 

# Side effect of the fix
The fix breaks existing code that is built around this bug. An example is `CGRect` constructor that stops working:  
```java
public CGRect(double x, double y, double width, double height) {
    getOrigin().setX(x).setY(y);
    getSize().setWidth(width).setHeight(height);
}
```

After the fix `getOrigin()`/`getSize()` return a copy (byValue) structures. And changes to them doesn't affect `CGRect` struct itself anymore.

