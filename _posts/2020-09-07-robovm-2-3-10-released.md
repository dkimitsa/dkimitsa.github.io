---
layout: post
title: 'RoboVM 2.3.10 release'
tags: [whatsnew]
---
Available at Maven and IDE plugins are available for [download](http://robovm.mobidevelop.com).

## Known issues
Release requires Xcode 11.4+, if running previous one following error is expected:
```
[ERROR] Undefined symbols for architecture arm64:
[ERROR]   "___darwin_check_fd_set_overflow", referenced from:
```

## What's changed
* support for iOS14 devices 
* iOS13.6 bindings
* build system updated to work on recent JDK versions (was working only on JDK8)
* compiler: generic class arguments and @Block parameters [pr419](https://github.com/MobiVM/robovm/pull/419)
* compiler: support for non-static @Bridge method in enums classes [pr420](https://github.com/MobiVM/robovm/pull/420)
* compiler: support for @Block member in structs [pr421](https://github.com/MobiVM/robovm/pull/421)
* fixed: compilation failed on @Bridge annotate covariant return synthetic method [pr422](https://github.com/MobiVM/robovm/pull/422)
* Support for Struct.offsetOf in structs [pr431](https://github.com/MobiVM/robovm/pull/431)
* workaround for missing objc classes(ObjCClassNotFoundException) [pr442](https://github.com/MobiVM/robovm/pull/442)
* added: experimental and formal bitcode support, [pr443](https://github.com/MobiVM/robovm/pull/442)
* fixed: `error code -34018` when using Security API on simulator: [pr447](https://github.com/MobiVM/robovm/pull/447)
* idea: fixed: annoying Android gradle faced [#242](https://github.com/MobiVM/robovm/issues/242)
* idea: migrated to gradle build system
* idea: "no Xcode dialog" is not blocking anymore [pr434](https://github.com/MobiVM/robovm/pull/434)
* idea: disabled `generate separate IDEA module per source set` [pr449](https://github.com/MobiVM/robovm/pull/449)
* debugger: various crash fixes (more stable now)
* added: workaround to support static libraries that use swift [pr474](https://github.com/MobiVM/robovm/pull/474)
* added: frameworkPath/extensionPath can be qualified (temporal support for xcframeworks) [pr484](https://github.com/MobiVM/robovm/pull/483)
* fixed: OOM on class with huge number of fields [pr485](https://github.com/MobiVM/robovm/pull/485)
* new icons for Idea plugin !

## Other notes
Idea plugin now is a `.zip` file and Safari will unpack it (by default) which will make it not functional. It has to be downloaded as `.zip` file and installed using `.zip` file, not with `.jar` file inside zip/unpacked folder.

Happy coding!
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).
