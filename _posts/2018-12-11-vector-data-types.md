---
layout: post
title: 'RoboVM and Vector data type. Why ARKit is not working'
tags: [bug]
---

User `pjwj` [reported problem](https://gitter.im/MobiVM/robovm?at=5be459f747217e07ffeee72e) when he was trying to use ARKit (first one?):
> Values in this matrix are not as I expect them to be, as translation components are very small (for example x*10^-35). Therefore virtual objects I'm adding to the AR scene are always positioned close to the origin.

Quick investigation shown that RoboVM is not able to handle vector data types such as `simd_float4x4` (mapped to `MatrixFloat4x4` in RoboVM) at compiler level. As result all vector data is passed incorrectly.  
Investigation and fix is bellow.
<!-- more -->  

## Way to reproduce
It is enough just to use following native code:
```c
@implementation VectorTestProvider
+(simd_float2x4) testFloat2x4 {
    simd_float2x4 r = {
        (simd_float4){1, 2, 3, 4},
        (simd_float4){8, 7, 5, 4}
    };
    NSLog(@"Returning simd_float2x4 { %f %f %f %f } { %f %f %f %f }",
          r.columns[0][0], r.columns[0][1], r.columns[0][2], r.columns[0][3],
          r.columns[1][0], r.columns[1][1], r.columns[1][2], r.columns[1][3]);
    return r;
}
@end
```

with following binding:  
```java
@Library(Library.INTERNAL)
@NativeClass
public class VectorTestProvider extends NSObject {
    @Method(selector = "testFloat2x4")
    native static @ByVal MatrixFloat2x4 testFloat2x4();
}
```

Calling `VectorTestProvider.testFloat2x4()` with snippet bellow crashes on simulator with GPF (x86_64) and provides garbage on device (ARM64).
```java
 MatrixFloat2x4 r = VectorTestProvider.testFloat2x4();
 ```

## Root case x86_64
RoboVM compiler generates c bridge methods for by value return types. For MatrixFloat2x4 it looks as bellow:
```c
void bridge(void* target, void* ret, void* self, void* sel) {
    typedef struct {float m0;float m1;float m2;float m3;} st1;
    typedef struct {float m0;float m1;float m2;float m3;} st2;
    typedef struct {st1 m0; st2 m1;} st;
    *((st*)ret) = ((st (*)(void*, void*)) target)(self, sel);
}
```

Where `st` is a struct to receive result and it is allocated on stack. `+(simd_float2x4) testFloat2x4` code crashes with GPF on following instruction:
![]({{ "/assets/2018/12/11/vectors_x86_64_crash.png" | absolute_url}})

`movaps %xmm0, 0x10(%rdi)` is SSE instruction that operates on 16 bytes aligned addresses but `rdi` contains `0x00007ffeefbfb8d8` address of allocated on stack `st` struct. It is not 16 by aligned and this causes `EXC_I386_GPFLT` exception.  
Reason for case: generic structs are subject for ABI specific alignment and it is 8 bytes for Amd64. Bridge shall generate vectorized struct with proper alignment.

## Root case ARM64
Beside alignment problem of `x86_64` there is a difference of how vector data being passed due calling conventions. [RetCC_AArch64_AAPCS](https://github.com/llvm-mirror/llvm/blob/48d92865105f5cb0e8c7bb146a31ccedfaccfda5/lib/Target/AArch64/AArch64CallingConvention.td#L92) specifies that vector data will be returned in registers as long as it fits there. But as long as bridge generates generic structs -- it will pass pointer and expect that result to be filled by it. As result it will not crash but bridge will not pick up returned data from registers.  
Here is illustration of vector and structure call of same sized data, setup if following:  
```objc
typedef struct { float m1, m2, m3, m4;} v4;
typedef struct { v4 m1, m2;} m2x4;

@interface test
+(simd_float2x4) t1;
+(m2x4) t2;
@end  
```

`[test t1]` (with vector data type) produce following IR code:
```
%4 = load %struct._class_t*, %struct._class_t** @"OBJC_CLASSLIST_REFERENCES_$_", align 8
%5 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_, align 8, !invariant.load !8
%6 = bitcast %struct._class_t* %4 to i8*
%7 = call %struct.simd_float2x4 bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to %struct.simd_float2x4 (i8*, i8*)*)(i8* %6, i8* %5)
```

`[test t2]` (with generic structs) produce following IR code:
```
%3 = alloca %struct.m2x4, align 4
%11 = load i8*, i8** @OBJC_SELECTOR_REFERENCES_.2, align 8, !invariant.load !8
%12 = bitcast %struct._class_t* %10 to i8*
call void bitcast (i8* (i8*, i8*, ...)* @objc_msgSend to void (%struct.m2x4*, i8*, i8*)*)(%struct.m2x4* sret %3, i8* %12, i8* %11)
```

And difference here as described above: generic structs passes pointer to memory to receive data. While data is returned by registers (t1 case).

## The fix
To fix this it is required to generate vectorized bridge method. Sample `bridge` method from `x86_64` case shall be generated as bellow:  
```c
void bridge(void* target, void* ret, void* p0, void* p1) {
    typedef __attribute__((__ext_vector_type__(4))) float st1;
    typedef struct { st1 m[2];} st;
    *((st*)ret) = ((st (*)(void*, void*)) target)(p0, p1);
}
```

It specifies `__ext_vector_type__` attribute which automatically makes it 16 byte aligned and applies vector data type calling convention.

## Pull request
These changes are available in [pull request #339](https://github.com/MobiVM/robovm/pull/339)  

Beside changing the way bridge method is generated PR also changes following:
* added special `Vectorized` annotation to mark structures as vector types. Only annotated structures will be processed in special way;
* in all IR code removed empty first member of structure that was reserved for inherited structures. If there is no super structure class -- there will be no entry;
* `Vectorized` structures also has proper IR code
