---
layout: post
title: 'bugfix #295 debugger case: crash due Kotlin data classes and broken Java classes'
tags: [fix, debugger, kotlin]
---
Debugger maintenance.  
**History** [Robovm compiler crashes with Kotlin modules when compiling for debug  #295](https://github.com/MobiVM/robovm/issues/295)  
**Fix** [PR296](https://github.com/MobiVM/robovm/pull/296)  

Fixed:
1. Crash when trying to debug application that contained Kotlin data modules (or any synthetics methods);  
2. Crash when starting debugger for binary that contains classes with error (e.g. class not found for dependencies)  

Details:
<!-- more -->
## Crash due Kotlin data classes
Root case -- is synthetic methods without debug information, as result `DebugInformationPlugin` was generating wrong line number in DWARF debug information, example:
```
define weak %Object* @"[J]com.mycompany.myapp.TestKotlin.component1()Ljava/lang/String;"(%Env* %p0, %Object* %p1) nounwind noinline optsize {
label0:
    %r0 = alloca %Object*
    %$r1 = alloca %Object*
    call void @"llvm.dbg.declare"(metadata %Object** %r0, metadata !19), !dbg !{i32 2147483647, i32 0, !17, null}
...
}    
```

`!dbg !{i32 2147483647, i32 0, !17, null}` contains `line number == 2147483647` which is invalid and caused crash in LLVM lib:
```
Thread 87 Crashed:: Java: pool-20-thread-3
0   libsystem_kernel.dylib              0x00007fff78fbab6e __pthread_kill + 10
1   libsystem_pthread.dylib             0x00007fff79185080 pthread_kill + 333
2   libsystem_c.dylib                   0x00007fff78f161ae abort + 127
3   libjvm.dylib                        0x000000010748faa8 os::abort(bool) + 22
4   libjvm.dylib                        0x0000000107598b2d VMError::report_and_die() + 2207
5   libjvm.dylib                        0x0000000107494116 JVM_handle_bsd_signal + 511
6   libjvm.dylib                        0x00000001074919dc signalHandler(int, __siginfo*, void*) + 45
7   libsystem_platform.dylib            0x00007fff79178f5a _sigtramp + 26
8   ???                                 0x00006080036f4b00 0 + 106102929705728
9   librobovm-llvm1310633165281695321.dylib     0x000000018be9950b llvm::DwarfDebug::collectDeadVariables() + 427
10  librobovm-llvm1310633165281695321.dylib     0x000000018be995bc llvm::DwarfDebug::finalizeModuleInfo() + 124
11  librobovm-llvm1310633165281695321.dylib     0x000000018be99d10 llvm::DwarfDebug::endModule() + 48
12  librobovm-llvm1310633165281695321.dylib     0x000000018be7f187 llvm::AsmPrinter::doFinalization(llvm::Module&) + 1847
13  librobovm-llvm1310633165281695321.dylib     0x000000018c508ae5 llvm::FPPassManager::doFinalization(llvm::Module&) + 53
14  librobovm-llvm1310633165281695321.dylib     0x000000018c508fec llvm::legacy::PassManagerImpl::run(llvm::Module&) + 1228
15  librobovm-llvm1310633165281695321.dylib     0x000000018b947c05 LLVMTargetMachineEmitToOutputStream + 421
16  librobovm-llvm1310633165281695321.dylib     0x000000018b946817 Java_org_robovm_llvm_binding_LLVMJNI_TargetMachineEmitToOutputStream + 71
17  ???                                 0x0000000116999cb2 0 + 4674133170
18  ???                                 0x000000011b27ae74 0 + 4750552692
```

Fix: is omit methods without debug information, check [commit](https://github.com/MobiVM/robovm/pull/296/commits/e6c05ad76eb898d2f9f3576bfb64d1d83bd1d820)

## Crash due broken Java classes
Such classes are included compilation time by RoboVM and marked as `has error`. During runtime this causes exceptions like NoClassDefFoundError. Debugger was crashing as it was incorrectly reading class meta data cache. In general it was accessing fields that are not present in case class is marked as `hasError`.   
Fix: check `hasError` status, [commit](https://github.com/MobiVM/robovm/pull/296/commits/f80dff54459c3a5cfd1668bc2d9ba8259e9423c1)
