---
layout: post
title: "Bro: Support for packed structures and ABI mystery"
tags: [bro-gen, bindings, hacking, structs]
---
Packed C structures were not supported what caused and [issue  #374](https://github.com/MobiVM/robovm/issues/374). Adding support for them seemed as a quick win but it had own pitfalls. As part of quick win following was done:

(code was delivered as [PR 378](https://github.com/MobiVM/robovm/pull/378))

## Added @Packed(align param) annotation to mark structures to be packed:
To mark structure as packed it is enough just annotate it with @Packed.  
Following C structure:
```c
#pragma pack(push, 2)
struct PascalString { short length; long long v;};
#pragma pack(pop)
```
is to be bind into following Java one:
```java
@Packed(2)
public class PascalString extends Struct<PascalString> {
    @StructMember(0) public native short length();
    @StructMember(0) public native PascalString length(short length);
    @StructMember(1) public native long v();
    @StructMember(1) public native PascalString v(long v);
}
```

## Bro compiler generates proper LLVM IR packed Java structures:
<!-- more -->
For java struct shown above following IR code is generated:
```
<{i16, i64}>
```

## Bro compiler generates proper C wrappers for @ByVal struct parameters and returns:  
As passing struct by value is quite tricky and architecture dependent RoboVM just generates C wrapper for such cases and leave this problem up to Clang, for Java struct defined above and java method that returns struct by value:
```java
@Method(selector = "test2")
public static native @ByVal PascalString test2();
```
following C wrapper code is generated:  
```c
void test2_wrap(void* target, void* ret, void* p0, void* p1) {
    #pragma pack(push, 2)
    typedef struct {short m0;long long m1;} test2_wrap_0000;
    #pragma pack(pop)
    *((test2_wrap_0000*)ret) = ((test2_wrap_0000 (*)(void*, void*)) target)(p0, p1);
}
```

## bro-gen -- automatic support not possible
It is not possible to detect `pragma pack` as `libclang` doesn't expose `MaxFieldAlignmentAttr` attribute. But clang itself expose it when dumping AST with following command line:  
> clang -emit-ast <filename>  

Another tool was written to automatically detect not configured packed structures: [find-pragma-pack.rb](https://github.com/dkimitsa/robovm-bro-gen/blob/master/find-pragma-pack.rb).  
Usage is pretty simple:  
> find-pragma-pack.rb foundation.yaml [other yaml]   

And it will report all structs that requires `@Packed` to be added into yaml. Example of yaml:  
```yaml
classes:
    # Structs
    GCMicroGamepadSnapShotDataV100:
        annotations: ['@Packed(1)']
```


**#### That is the end of relatively quick win**

# Problems
Code was working for set tests but was as well crashing on another relatively simple sample java sample above. Root case for this was wrong logic in selecting bridge function for methods that return structures by value.  

**Background:**
Platform ABI specifies how structures to be returned: either by pointer or in register. The rules depend on target, structure type and not declared ideas. When calling generic C function Clang will handle the way to receive struct returned by value. Thats why C wrapper is generated. But for calling 'Objective C' selector proper ObjC routine to be used:  
* `objc_msgSend` -- if struct is returned in registers;
* `objc_msgSend_stret` -- if struct is returned by pointer.  

There is already logic in place in `ObjCRuntime.isStret()` that covered generic cases that were working for years (for unpacked structures). To find out what is wrong it is required to review calling conventions for every platform RoboVM targets.

# ABI mysteries

## ARM64
For this platform there is no `objc_msgSend_stret` so issue doesn't exist. What a quick win.

## ARMv7
ABI is defined in [Procedure Call Standard for the ARM Architecture (AAPCS)](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf) and it says:
> A Composite Type not larger than 4 bytes is returned in r0

`ObjCRuntime.isStret()` handles it as bellow:  
```java
if (Bro.IS_ARM && Bro.IS_32BIT) {
   if (structSize > 4) {
       // On ARM stret has to be used for structs
       // larger than 4 bytes
       return true;
   }
}
```

In real(clang) word things are bit different. Truth is in Clang sources, method `ARMABIInfo::classifyReturnType`. It says:
> // Integer like structures are returned in r0.

This means that only allowed structs are structs that contain only ONE integer like member.
Even for not packed structures `ObjCRuntime.isStret()` was returning wrong result.

## x86
ABI is defined in [System V Application Binary Interface: Intel386 Architecture Processor Supplement](http://www.sco.com/developers/devspecs/abi386-4.pdf) and it says:
> Aggregate types (structs and unions) are always returned in memory.

But `ObjCRuntime.isStret()` handles it as bellow:  
```java
if (Bro.IS_X86 && Bro.IS_32BIT) {
    if (structSize > 2 && structSize != 4 && structSize != 8) {
        // On x86 stret has to be used for all structs except
        // of size 1, 2, 4 and 8 bytes.
        return true;
    }
}
```

While Apple refers ABI it also [declares following](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/LowLevelABI/130-IA-32_Function_Calling_Conventions/IA32.html#//apple_ref/doc/uid/TP40002492-SW5):  
> Structures. The called function returns structures according to their aligned size.
Structures 1 or 2 bytes in size are placed in EAX.
Structures 4 or 8 bytes in size are placed in: EAX and EDX.

That is against ABI but tests show that this is true. Packed/unpacked. As long as size of struct match conditions -- `ObjCRuntime.isStret()` returns correct result here.

## x86_64
This Arch was causing crashes during testing.  
ABI is defined in [System V Application Binary Interface AMD64 Architecture Processor Supplement](https://github.com/hjl-tools/x86-psABI/wiki/X86-psABI) and it says:
> If the size of an object is larger than four eight bytes, or it contains unaligned fields, it has class MEMORY
>...
> The post merger clean up described later ensures that, for the processors that do not support the m256 type, if the size of an object is larger than two eightbytes and the first eightbyte is not SSE or any other eightbyte is not SSEUP, it still has class MEMORY.

```java
`ObjCRuntime.isStret()` handles it as bellow:  
if (Bro.IS_X86 && Bro.IS_64BIT) {
    if (structSize > 16) {
        return true;
    }
}
```
which is ok for non-packed structs (as they are always aligned) but it case of packed additional rule to be added.


# The fix
It is quite not effective to find out in runtime if structure has all fields aligned (for x86_64) or if it is `integer like structure`(for armv7). Support for this case was added to bro-compiler and runtime:
- bro compiler find outs and for each structs setups flags: 1) if it `unaligned` and 2) if it `non integer like`;
- bro gen adds synthetic method `$attr$stretMetadata` for structure classes which returns bitmask of these flags;
- `ObjCRuntime.isStret()` will use these flags to solve described above corner cases.

 In result `ObjCRuntime.isStret()` was modified as bellow:
 ```java
 ...
} else if (Bro.IS_X86 && Bro.IS_64BIT) {
   if (structSize > 16) {
       return true;
   }

   // System V Application Binary Interface AMD64 Architecture Processor Supplement.
   // 3.2.3: If it contains un- aligned fields, it has class MEMORY
   int attr = getStructAttributes(returnType);
   if ((attr & ATTR_UNALIGNED) != 0) {
       return true;
   }
} else if (Bro.IS_ARM && Bro.IS_32BIT) {
   if (structSize > 4) {
       // On ARM stret has to be used for structs
       // larger than 4 bytes
       return true;
   }

   // only Integer like structures are returned in r0.
   int attr = getStructAttributes(returnType);
   if ((attr & ATTR_NOT_SINGLE_INT_STRUCT) != 0) {
       return true;
   }
}
...
```

## Other
Tests that generates LLVM IR and displays if `objc_msgSend_stret` or `objc_msgSend` is used for different architectures and structs is pushed to [codesnipets/clang_stret_test](https://github.com/dkimitsa/codesnippets/tree/master/clang_stret_test)
