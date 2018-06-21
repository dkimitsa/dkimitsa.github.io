---
layout: post
title: 'RoboVM 2.3.4 released, what''s new'
tags: [whatsnew]
---
It took a while for this release, compiler and gradle plugging is already on Maven and plugins will be available soon for [download](http://robovm.mobidevelop.com/downloads/releases/idea/).  


* What's new
    - [Added support for 'App Extensions': still can't create one with RoboVM but allows to use existing one]({{ site.baseurl }}{% post_url 2018-01-18-tutorial-using-app-extensions-with-robovm %}) ([PR255](https://github.com/MobiVM/robovm/pull/255))
    - [Framework target was improved that allows writing framework out of box without need of native coding]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %}) ([PR253](https://github.com/MobiVM/robovm/pull/253))
    - [regenerated embedded icu51.dat to fix crashes on several locales]({{ site.baseurl }}{% post_url 2017-10-25-icudt51l.dat-crash-on-some-locales %}) ([PR227](https://github.com/MobiVM/robovm/pull/227))
    - Enabling Maven debug options ([#254](https://github.com/MobiVM/robovm/issues/254), [PR258](https://github.com/MobiVM/robovm/pull/258))
    - [ios11.3]({{ site.baseurl }}{% post_url 2018-04-03-ios-11-3-binding %}) ([PR280](https://github.com/MobiVM/robovm/pull/280)) and [ios11.4]({{ site.baseurl }}{% post_url 2018-05-30-ios-11-4-binding %}) ([PR299](https://github.com/MobiVM/robovm/pull/299)) bindings
    - Resurrect junit support ([PR281](https://github.com/MobiVM/robovm/pull/281))
    - [Gradle pluggin maintaince: support for debugger, dependency to build, clean cache etc]({{ site.baseurl }}{% post_url 2018-05-10-gradle-plugin-maintenance %}) ([PR292](https://github.com/MobiVM/robovm/pull/292))


* bugfixes
    - CBAdvertisementData cannot be instantiated]([#224](https://github.com/MobiVM/robovm/issues/224))
    - [RoboVM doesn't strips all simulator archs from dynamic framework which causes ITMS-90087 during submission]({{ site.baseurl }}{% post_url 2017-10-27-fix229-itms-90087 %}) ([#228](https://github.com/MobiVM/robovm/issues/228), [PR229](https://github.com/MobiVM/robovm/pull/229))
    - [IB integrator generates wrong class name for empty @CustomClass, doesn't support structs in IBInspectable]({{ site.baseurl }}{% post_url 2017-11-03-fix230-interface-builder-and-struct-outlets %}) ([#230](https://github.com/MobiVM/robovm/issues/230), [PR231](https://github.com/MobiVM/robovm/pull/231))
    - [debugger: fixed debugger crash when resolving stack variables on arm64]({{ site.baseurl }}{% post_url 2017-12-06-debugger-stack-var-arm64-fix239 %}) ([#239](https://github.com/MobiVM/robovm/issues/239), [PR240](https://github.com/MobiVM/robovm/pull/240))
    - robovm doesn't copies all swift libraries ([#244](https://github.com/MobiVM/robovm/issues/244), [PR245](https://github.com/MobiVM/robovm/pull/245))
    - InterfaceBuilderClassesPlugin complains on missing classes in xib but they are not ([#246](https://github.com/MobiVM/robovm/issues/246), [PR247](https://github.com/MobiVM/robovm/pull/247))
    - [Debugger hangs when evaluating native objects that has no corresponding Java class loaded]({{ site.baseurl }}{% post_url 2018-01-19-debugger-runtime-class-resolution-fix %}) ([PR256](https://github.com/MobiVM/robovm/pull/256))
    - [Debugger - broken step over and crashed when using lambda]({{ site.baseurl }}{% post_url 2018-01-22-debugger-stepping-and-labda-fixes %}) ([PR257](https://github.com/MobiVM/robovm/pull/257))
    - [robovm + idea: wrong simulator version is launched than selected in run configuration]({{ site.baseurl }}{% post_url 2018-02-07-fix263-simulator-selection-doesnt-work-idea %}) ([#262](https://github.com/MobiVM/robovm/issues/262), [PR263](https://github.com/MobiVM/robovm/pull/263))
    - [added flush as Unhandled exception will not be directed to NSLog otherwise]({{ site.baseurl }}{% post_url 2018-02-12-investigating-silent-crash %}) ([PR267](https://github.com/MobiVM/robovm/pull/267))
    - [Gradle: Unable to start 12.5 inch iPad simulator]({{ site.baseurl }}{% post_url 2018-02-15-fix233-simulator-selection-doesnt-work-gradle %}) ([#233](https://github.com/MobiVM/robovm/issues/233), [PR273](https://github.com/MobiVM/robovm/pull/273))
    - [dSYM generation heavily delayed on XCode 9.3]({{ site.baseurl }}{% post_url 2018-04-04-fix279-extremely-slow-dsymutil %}) ([#279](https://github.com/MobiVM/robovm/issues/279), [PR282](https://github.com/MobiVM/robovm/pull/282))
    - [IDEA: Install failure disables run button]({{ site.baseurl }}{% post_url 2018-04-30-fix286-fix-285-clear-cache-on-new-snapshot %}) ([#285](https://github.com/MobiVM/robovm/issues/285), [PR289](https://github.com/MobiVM/robovm/pull/289))
    - [Signing app ext and frameworks for Simulator: Apps won't run on simulator if embedded frameworks are present]({{ site.baseurl }}{% post_url 2018-04-30-fix286-fix-285-clear-cache-on-new-snapshot %}) ([#286](https://github.com/MobiVM/robovm/issues/286), [PR289](https://github.com/MobiVM/robovm/pull/289))
    - [Fixed code sign auto signature/provisioning profile pickup]({{ site.baseurl }}{% post_url 2018-05-16-fix-app-ext-autoprofile-bro-gen %}) ([PR293](https://github.com/MobiVM/robovm/pull/293))
    - [Robovm compiler crashes with Kotlin modules when compiling for debug]({{ site.baseurl }}{% post_url 2018-05-23-fix295-crashes-kotlin-debugger %}) ([#295](https://github.com/MobiVM/robovm/issues/295), [PR296](https://github.com/MobiVM/robovm/pull/296))
    - [fixed issue when breakpoint were not working in kotlin]({{ site.baseurl }}{% post_url 2018-05-25-fixing-debugger-kotlin-case %})  ([PR297](https://github.com/MobiVM/robovm/pull/297))
    - [fix for "WARNING ITMS-90725: "SDK Version Issue]({{ site.baseurl }}{% post_url 2018-06-01-fixing-itms-90725 %}) ([#302](https://github.com/MobiVM/robovm/issues/302), [PR301](https://github.com/MobiVM/robovm/pull/301))
    - [TLSv1.2 handshake fixed]({{ site.baseurl }}{% post_url 2018-06-13-fix306-ssl-handshake-error %}) ([#306](https://github.com/MobiVM/robovm/issues/306), [PR308](https://github.com/MobiVM/robovm/pull/308))

* Useful tutorials
    - [Tutorial: using app-extensions in RoboVM iOS application]({{ site.baseurl }}{% post_url 2018-01-18-tutorial-using-app-extensions-with-robovm %})
    - [Tutorial: writing Framework using improved framework target]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %})
    - [Tutorial: Crash Reporters and java exceptions]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %})
    - [Tutorial: Investigating silent crash of Release build (e.g. Port death watcher fired)]({{ site.baseurl }}{% post_url 2018-02-12-investigating-silent-crash %})
    - [Tutorial: Interface Builder integration in RoboVM]({{ site.baseurl }}{% post_url 2017-10-24-xcode-ib %})
    - [Tutorial: using bro-gen to quick generate bindings]({{ site.baseurl }}{% post_url 2017-10-19-bro-gen-tutorial %})
