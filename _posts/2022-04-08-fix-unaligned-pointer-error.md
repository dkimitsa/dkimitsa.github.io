---
layout: post
title: 'Fixing: ld: unaligned pointer(s) for architecture arm64'
tags: [fix, m1]
---

Xcode started complaining about unaligned pointer in RoboVM binaries since 8.3 ([issue #123](https://github.com/MobiVM/robovm/issues/123)) but these warnings (altogether with other possible) were suppressed with `-w` option. 
But with Xcode13.3 it started failing with `error ld: unaligned pointer(s) for architecture arm64` on some projects (while compiled ok but probably with changes to it might fail).   

# Trying to reproduce 
<!-- more -->
Quickest way is to get confirmation from Xcode world not related to RoboVM. Following snippet was used for test:  
```c
typedef struct __attribute__((packed)) {
    char a;
    char *c;
} mystruct;


int main() {
  mystruct aa = {0, "unalligned"};
  return sizeof(aa);
}
```

## Compiling for iphone/arm64
> clang --target=arm64-apple-ios -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS15.4.sdk/ ./1.c

result: `Works!` and this is `STRANGE` 

## Compiling for simulator/m1-arm64
> clang --target=arm64-apple-ios-simulator -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator15.4.sdk/ ./1.c\
>ld: warning: pointer not aligned at address 0x100004001 (l___const.main.aa + 1 from /var/folders/vr/vkkqdlys4mq0nvfnf_mwnfh5g3f__x/T/1-d8cc47.o)
>ld: unaligned pointer(s) for architecture arm64
>clang: error: linker command failed with exit code 1 (use -v to see invocation)

Result: `Fails!`

## Compiling for console/m1-arm64
>clang --target=arm64-apple-macosx -isysroot /Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk/ ./1.c
>ld: warning: pointer not aligned at address 0x100004001 (l___const.main.aa + 1 from /var/folders/vr/vkkqdlys4mq0nvfnf_mwnfh5g3f__x/T/1-c85adc.o)
>ld: unaligned pointer(s) for architecture arm64
>clang: error: linker command failed with exit code 1 (use -v to see invocation)

Result: `Fails!`

## Bottom line 
Results a bit strange considering that compilation is done against same `arm64` CPU arch and its might be as bug in XCode and new world requirement, and it's better to fix it.
One of the possible reasons for these requirements is `LC_DYLD_CHAINED_FIXUPS` mach-o command that Apple started using with iOS binaries and to fix addresses these should be properly aligned. 

# The fix 
## Root case and approach  
RoboVM writes metadata `[classinfo]` as well as `[attributes]` for class/method/field in way of packed structs and this causes alignment issues in case of pointers.   
The way to fix it is to migrate from `packed` to `regular` structs. This will require adapting:
- metadata struct generation code;
- VM runtime to read data on field boundaries;
- Debugger code that reads this metadata to know info about classes.
 
Most changes are trivial but there is a trick. 

## Long story short -- what will it cost
As unpacked struct have gaps it will cost extra memory for it. Sample run shows `25%` bigger memory footprint for these metadatas (`5525197` to `6935824` bytes).  

## The trick: inner structs and their alignment
As data was packed -- VM just read data `field by field` by moving pointer. In case not packed there padding being added to keep each member aligned to its layout boundary. 
The moment comes for inner struct members as:
> For struct, other than the alignment need for each individual member, 
> the size of whole struct itself will be aligned to a size divisible by 
> size of the largest individual member, by padding at end

### Bright example: 
```c
typedef struct  {
    uint8_t kind;
    struct {
       uint32_t kind1_field1;
       uint16_t kind1_field2;
    };
} mystruct1;

typedef struct  {
    uint8_t kind;
    struct {
       uint32_t kind1_field1;
       uint64_t kind2_field2;
    };
} mystruct2;
```

In both cases after `kind` field `int32` comes, and it is expected to be at int32 boundary but alignment for it will be directed by biggest member of struct it belongs. And its `int64`:
```
l___const.main.st2:
	.byte	2                               ## 0x2
	.space	7
	.long	3                               ## 0x3
	.space	4
	.quad	4                               ## 0x4
```

As result this breaks existing parsing. 

## But there is a trick:
The trick is to make structure flat in way that it doesn't contain any other struct members: 
```c
typedef struct  {
    uint8_t kind;
    uint32_t kind1_field1;
    uint64_t kind2_field2;
} mystruct2;
```

Will compile to pretty expected one:
```
L___const.main.st2:
	.byte	2                               ## 0x2
	.space	3
	.long	3                               ## 0x3
	.quad	4                               ## 0x4
```
# Code 
The fix was delivered as [PR639](https://github.com/MobiVM/robovm/pull/639)
