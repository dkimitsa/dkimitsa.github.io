---
layout: post
title: 'Debugger: support for binaries with chained fixup'
tags: [fix, debugger, ios15, dyld]
---
Xcode13.3 bring [unaligned pointer problem]({{ site.baseurl }}{% post_url 2022-04-08-fix-unaligned-pointer-error %}) during linking but also debugger launch was crashing with: 
>[ERROR] Couldn't start application
>java.lang.IllegalArgumentException: there is no region for addr @10000101effaa0
>	at org.robovm.debugger.utils.bytebuffer.CompositeDataBuffer.setPosition(CompositeDataBuffer.java:37)

`0x10000101effaa0` indeed looks weird for memory address, even looks like [tagged pointer](https://developer.apple.com/videos/play/wwdc2020/10163/) at first (but not). Anyway it introduced a level of complication that is not allowing debugger to read internal RoboVM structures like class info anymore. 

# LC_DYLD_CHAINED_FIXUPS and LC_DYLD_EXPORTS_TRIE
<!-- more -->
>Apple:
> All programs and dylibs built with a deployment target of macOS 12 or 
> iOS 15 or later now use the chained fixups format. This uses different 
> load commands and LINKEDIT data, and won’t run or load on older OS versions.

As usual there is not much of technical details. Nice post by `Noah Martinz` [How iOS 15 makes your app launch faster](https://www.emergetools.com/blog/posts/iOS15LaunchTime) light things a little but not enough to consider things clear or as technical spec. 
Lucky for as Apple releases [dyld](https://opensource.apple.com/tarballs/dyld/) sources as past of [Open Source at Apple](https://opensource.apple.com), and it is the only source of truth here.

Long story short -- `dyld` now organizes address bind/rebase as part of chained data and keep this chains inside pointers to be fixes.
That is a root of strange pointers like `@10000101effaa0`.

RoboVM debugger operates with binary as it saved on file system we have to perform chained fixup to get proper pointer values to be able to read internal RoboVM structures required for debugger.

# Approach 

## How to implement:
There is a lack of spec and doing same as in [dyld](https://opensource.apple.com/tarballs/dyld/) is good option. 
Mostly work to be done it is port of `MachOLoaded::fixupAllChainedFixups` methods and its sequence calls. 
In general things to be done are pretty simple -- follow pointer chain, check bits, build new pointer value and put it back.

## Fixing pointers
Another moment -- is how to fix pointer. One approach would be is to create a `remap` hashmap and look for values each time.
But amount of remap locations is huge enough even for simple project.  Second approach is to use `FileChannel.MapMode.PRIVATE`(copy-on-write)  when mapping file channel to ByteBuffer and just fix pointer over file.
This will allow keeping debugger logic not affected but only image loading. And this approach to be taken.

### Validation
There is `dyldinfo` that shows useful information when called with `-rebase -bind`. 
But `dyld` sources contains part for `dyld_info` which has extra command line options to be useful (and missing in bundled with Xcode `dyldinfo`): `-fixup_chains`, `-fixup_chain_details`. 

`dyld_info` to be build from sources that will allow to see extra debug output, compare results with own implementation. Also debug it to see how it works its a good option.\
Check bonus part for compilation instructions.

# Code 
The fix was proposed as [PR645](https://github.com/MobiVM/robovm/pull/645)


# Bonus -- building `dyld_info`
Its a bit tricky as depends on `macosx.internal` and some headers that are missing. 

1. Download and unpack [dyld-852.2.tar.gz](https://opensource.apple.com/tarballs/dyld/dyld-852.2.tar.gz)
2. Lucky for us, it contains `dyld.xcodeproj`. Open it;
3. Select `dyld_info` scheme;
      ![]({{ "/assets/2022/04/16/dyldinfo_project_scheme.png"}})

4. Build it, and it will fail with error:
> unable to find sdk 'macosx.internal'

5. Open project `dyld` settings -> `Build Settings` and change `Base SDK` from `macosx.internal` to `macOS`:
   ![]({{ "/assets/2022/04/16/dyldinfo_project_sdk.png"}})

6. Build it, and it will fail with complaint due missing headers like bellow:
> fatal error: 'corecrypto/ccdigest.h' file not found  
> fatal error: 'sandbox/private.h' file not found

7. To fix this and other missing header issues, prepare location where these will be provided:
- Create `${dyld-852.2}/external/` folder where all missing headers will be located
- Create a new `Configuration Setting File` with a name `dyld_info` in `configs` folder of Xcode project;
- Put reference to `external/` to `dyld_info.xcconfig`:
```
HEADER_SEARCH_PATHS=$(inherited) external/
```
- configure `dyld_info` to use this config:
  ![]({{ "/assets/2022/04/16/dyldinfo_target_config.png"}})

8. Missing libraries can be picked there:
- `corecrypto/` headers are present in [xnu-7195.81.3](https://opensource.apple.com/tarballs/xnu/). Copy `xnu-7195.81.3.tar.gz/EXTERNAL_HEADERS/corecrypto/` to '${dyld-852.2}/external/corecrypto/`;
- `_simple.h` is present in [Libc-825.40.1](https://opensource.apple.com/tarballs/Libc/). Copy `Libc-825.40.1.tar.gz/gen/_simple.h` to `${dyld-852.2}/external/_simple.h`
- `libc_private.h` is present in [Libc-1439.40.11](https://opensource.apple.com/tarballs/Libc/). Copy `Libc-1439.40.11.tar.gz/darwin/libc_private.h` to `${dyld-852.2}/external/libc_private.h`
- `sandbox/private.h` can be picked from [github/OSXPrivateSDK](https://github.com/samdmarshall/OSXPrivateSDK/blob/master/PrivateSDK10.10.sparse.sdk/usr/local/include/sandbox/private.h) and put it to `${dyld-852.2}/external/include/sandbox/private.h`

9. Then it will be massively complains about missing `bridgeos` in different `API_AVAILABLE` macros and simples way is to redefine `bridgeos` to something that is known to toolchain, like `watchos` (don't care for it in this build). `dyld_info.xcconfig` takes the final look:
```
HEADER_SEARCH_PATHS = $(inherited) external/
GCC_PREPROCESSOR_DEFINITIONS = $(inherited) bridgeos=watchos
```

Compilation will success with some amount code level warnings. Binary can be run with required new params and debuggedm what was required.
