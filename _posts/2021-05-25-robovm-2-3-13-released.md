---
layout: post
title: 'RoboVM 2.3.13 release and 2.3.14-SNAPSHOT'
tags: [whatsnew]
---
Available at [Idea Marketplace](https://plugins.jetbrains.com/plugin/14440-mobivm) (for AndroidStudio as well), also IDE plugins are available for [download](http://robovm.mobidevelop.com).

## 2.3.13: What's new
Fixes and maintenance release:    
* fixed: NumberFormatException when compiling 1.4/1.5 kotlin code for Debugger [PR581](https://github.com/MobiVM/robovm/pull/581)
* changed: the way how dynamic libraries appears at linker command line [PR580](https://github.com/MobiVM/robovm/pull/580)
* fixed #561: reverted binding of structs with flexible array members [PR565](https://github.com/MobiVM/robovm/pull/565)
* fixed: GlobalValueEnumeration crash #567 [PR571](https://github.com/MobiVM/robovm/pull/571)
* reworked way swift dependencies are picked up [PR552](https://github.com/MobiVM/robovm/pull/552)
* Idea plugin maintenance: support for 2021.1 EAP [PR570](https://github.com/MobiVM/robovm/pull/570)
* fix for #557. merged several CryptoLib/OpenSSL fixes [PR564](https://github.com/MobiVM/robovm/pull/564)
* fix for: ClassCastException - passing java Interface implementation as ObjC protocol [PR563](https://github.com/MobiVM/robovm/pull/563)
* hot fix for CompilerException while compiling UIKey.keyCode [PR553](https://github.com/MobiVM/robovm/pull/553)
* CocoaTouch 14.3 and fixes [PR551](https://github.com/MobiVM/robovm/pull/551)
* fixed: @ByVal is not working for GlobalValues/Struct getters [PR550](https://github.com/MobiVM/robovm/pull/550)
* fixed #542: Dagger with Kotlin, IDE IPA Creation Problem [PR549](https://github.com/MobiVM/robovm/pull/549)

## 2.3.14-SNAPSHOT:
Includes functionality currently in testing:  
* iOS14.5 bindings[PR575](https://github.com/MobiVM/robovm/pull/575)

Happy coding!  
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).
