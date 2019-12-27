---
layout: post
title: "iOS13 PostFix #5: implementation of missing Struct.offsetOf"
tags: ["fix", "compiler", "postfix"]
---
This post continues the series of fixes discovered during compilation of CocoaTouch library and improvements to compiler.
# PostFix #5: `Struct.offsetOf` always returns zero

Other postfixes:
* [PostFix #1: Generic class arguments and @Block parameters]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %})
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})
* [PostFix #7: bindings for ios13.2]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %})
* [PostFix #8: workaround for missing objc classes(ObjCClassNotFoundException)]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %})
* [PostFix #9: experimental and formal bitcode support]({{ site.baseurl }}{% post_url 2019-12-26-cocoa13-postfix9-formal-bitcode-support %})

`Struct.offsetOf` is required to proper implement initialization of variable size structs with flexible array member such as:
```c
struct vectord {
    size_t len;
    double arr[]; // the flexible array member must be last
};
```

## Root case and fix
`Struct` has `offsetOf` definition and it returns always zero similar to `sizeOf` method. `` its implemenation is being synthesized by RoboVM compiler and invocation of `sizeOf` being fixed during trampoline phase from `Struct.sizeOf` to `DestStruct.sizeOf` (DestStruct -- an example struct implementation).
Root case -- compiler doesn't not synthesize `offsetOf`.
<!-- more -->

## Obtaining memeber offset
Structure member offsset is to retrieved from prepared LLVM structure definition using `LLVMOffsetOfElement`. Code is simple -- just need to call it for eatch member:
```java
public int[] getStructMemberOffsets(StructureType structType) {
    // get offset of each struct member by calling llvm api
    int membersCount = structType.getTypeCount() - structType.getOwnMembersOffset();
    int offset = structType.getOwnMembersOffset(); // inherited struct (if any) goes as member 0, own members starts 1
    int[] offsets = new int[membersCount];
    for (int idx = 0; idx < membersCount; idx++)
        offsets[idx] = config.getDataLayout().getOffsetOfElement(structType, offset + idx);
    return offsets;
}
```

## Synthesize `offsetOf`
`offsetOf` is simple function of kind: `switch(memberIdx) -> return offset`. All constants are pre-calculated during compilation and funciton just returns constants. Code that synthesize `offsetOf` looks like bellow:
```java
private Function structOffsetOf(ModuleBuilder moduleBuilder, SootMethod method) {
    Function fn = createMethodFunction(method);
    moduleBuilder.addFunction(fn);

    int[] offsets = getStructMemberOffsets(structType);
    if (offsets.length > 0 ) {
        // function code -- basic switch of returns
        Label[] switchLabels = new Label[offsets.length];
        Map<IntegerConstant, BasicBlockRef> targets = new HashMap<IntegerConstant, BasicBlockRef>();
        for (int idx = 0; idx < offsets.length; idx++) {
            targets.put(new IntegerConstant(idx), fn.newBasicBlockRef(switchLabels[idx] = new Label(idx)));
        }

        Value idxValue = fn.getParameterRef(1); // 'env' is parameter 0
        Label def = new Label(-1);
        fn.add(new Switch(idxValue, fn.newBasicBlockRef(def), targets));

        // cases
        for (int idx = 0; idx < offsets.length; idx++) {
            fn.newBasicBlock(switchLabels[idx]);
            fn.add(new Ret(new IntegerConstant(offsets[idx])));
        }

        // default case -- array out of bounds exception
        fn.newBasicBlock(def);
    }
    Functions.call(fn, Functions.BC_THROW_ARRAY_INDEX_OUT_OF_BOUNDS_EXCEPTION, fn.getParameterRef(0),
        new IntegerConstant(offsets.length), fn.getParameterRef(1));
    fn.add(new Unreachable());

    return fn;
}
```

It produces `offsetOf` LLVM IR code similar to bellow:
```
define weak i32 @"[J]com.example.Testic5.offsetOf(I)I"(%Env* %p0, i32 %p1) nounwind noinline optsize {
label0:
    switch i32 %p1, label %label3 [ i32 0, label %label1 i32 1, label %label2 ]
label1:
    ret i32 0
label2:
    ret i32 8
label3:
    call void @"_bcThrowArrayIndexOutOfBoundsException"(%Env* %p0, i32 2, i32 %p1)
    unreachable
}
```

## Trampoline for `offsetOf`
For every `MyStruct.offsetOf` javac generates call to `Struct.offsetOf` and last is always return zero. `offsetOf` was synthesized for `MyStruct` with changes above but this happened after javac and it resolve `offsetOf` to `Struct` as it was not hidden in `MyStruct`. Solution for this is ultimately trampoline all `offsetOf` calls to final struct.
Same is already being done for `sizeOf` method in `TrampolineCompiler.java`:

## Source code
The fix was delivered as [PR431](https://github.com/MobiVM/robovm/pull/431)

