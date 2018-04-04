---
layout: post
title: 'bugfix #279: Extremely slow dsymutil after Xcode 9.3 update'
tags: [fix, xcode]
---
**History** [dSYM generation heavily delayed on XCode 9.3](https://github.com/MobiVM/robovm/issues/279)  
**Fix** [PR282](https://github.com/MobiVM/robovm/pull/282)  

Each Xcode update breaks something in RoboVM. This time it slowed down `dsymutil` that makes build time up to 30 min long and development process completely unusable. Small numbers on empty project, will use i386 slice from binary generated from empty framework sample:

With 9.3 dsymutil - **1 min 42 secs** vs **6 secs**:
```
> time (dsymutil -o test.dsym i386.bin)
real	1m42.932s
user	1m30.420s
sys	0m9.043s
```

With llvm-dsymutil (built it from source):
```
> time (./llvm-dsymutil -o test.dsym i386.bin)
real	0m5.813s
user	0m1.953s
sys	0m3.599s
```

Had no idea where to start from so started from old known bug [robovm/#1126](https://github.com/robovm/robovm/issues/1126). In general it was complaining that it was not able to find out object file for about 90K+ symbols:   
<!-- more -->
```
...
warning: (i386)  could not find object file symbol for symbol _[j]com.mycompany.myframework.Api$Calculator[ldcext]
warning: (i386)  could not find object file symbol for symbol _str_Lorg_2Frobovm_2Fobjc_2Fannotation_2FMethod_3B_00
warning: (i386)  could not find object file symbol for symbol _str_sub_00
warning: (i386)  could not find object file symbol for symbol _str_result_00
warning: (i386)  could not find object file symbol for symbol _str_org_2Frobovm_2Fobjc_2FObjCProtocol_00
warning: (i386)  could not find object file symbol for symbol _[j]com.mycompany.myframework.Api$Calculator[infostruct]
warning: (i386)  could not find object file symbol for symbol _[J]com.mycompany.myframework.Api$Calculator$ObjCProxy.reset()I
warning: (i386)  could not find object file symbol for symbol _[J]com.mycompany.myframework.Api$Calculator$ObjCProxy.add(I)I
warning: (i386)  could not find object file symbol for symbol _[J]com.mycompany.myframework.Api$Calculator$ObjCProxy.sub(I)I
warning: (i386)  could not find object file symbol for symbol _[J]com.mycompany.myframework.Api$Calculator$ObjCProxy.result()I
...
```

Digging [dsymutil source](https://github.com/llvm-mirror/llvm/blob/67eb8fdbb03eccd68274aa56abe1002d99f3bfc8/tools/dsymutil/MachODebugMapParser.cpp#L357) shown that these data comes from N_OSO type of symbol entries:
>#define	N_OSO	0x66	// object file name: name,,0,0,st_mtime

And these were really missing in binary:
```
> dsymutil -symtab i386.bin | grep N_OSO
...
... only library presents and no .o files
...
[2] 000018ba 66 (N_OSO) 03     0001   000000005a58b52b '~/projects/git/robovm/compiler/vm/target/binaries/ios/x86/librobovm-rt-dbg.a(rt.c.o)'
...
```

These are missing due fact that linker was not putting this information to it. Checking the source for last available Apple LD64 resulted [logic for these entries](https://github.com/tpoechtrager/cctools-port/blob/22ebe727a5cdc21059d45313cf52b4882157f6f0/cctools/ld64/src/ld/OutputFile.cpp#L5306):  

## Reason
Long story short: when linking it uses DWARF debug information: `AT_name` and `AT_comp_dir` to produce `N_OSO`. And in RoboVM it was wrong due following:
1. DebugInformationPlugin.java: location `of AT_name` and `AT_comp_dir` was swapped so it resulted in invalid path during linking which causes these names to be rejected;
2. For lambdas and all internal classes it was storing same base class name which caused `N_OSO` not to be generated as `AT_name` was not changed;

Example of broken DWARF:
```
>dwarfdump ~/.robovm/cache/ios/x86_64/release/Users/dkimitsa/projects/git/robovm/compiler/rt/target/robovm-rt-2.3.4-SNAPSHOT.jar/dalvik/system/BlockGuard$1.class.o
----------------------------------------------------------------------
 File: ~/.robovm/cache/ios/x86_64/release/Users/dkimitsa/projects/git/robovm/compiler/rt/target/robovm-rt-2.3.4-SNAPSHOT.jar/dalvik/system/BlockGuard$1.class.o (x86_64)
----------------------------------------------------------------------
.debug_info contents:
0x00000000: Compile Unit: length = 0x00000094  version = 0x0002  abbr_offset = 0x00000000  addr_size = 0x08  (next CU at 0x00000098)
0x0000000b: TAG_compile_unit [1] *
             AT_producer( "RoboVM 2.3.4-SNAPSHOT" )
             AT_language( DW_LANG_Java )
             AT_name( "/" )
             AT_stmt_list( 0x00000000 )
             AT_comp_dir( "BlockGuard$1.java" )
             AT_low_pc( 0x00000000000002e3 )
             AT_high_pc( 0x0000000000000310 )
```


## Fix
Make things right -- check [PR282](https://github.com/MobiVM/robovm/pull/282)  
