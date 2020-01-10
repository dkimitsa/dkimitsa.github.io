---
layout: post
title: 'RoboVM 2.3.10-SNAPSHOT, experimental build'
tags: [whatsnew]
---
@Tomski made first 2.3.10-SNAPSHOT build today. It artifacs deployed to sonatatype. Idea snapshot plugin is available for [download](http://robovm.mobidevelop.com/downloads/snapshots/idea).


*PLEASE NOTE:*
* *IMPORTANT:* Idea plugin is now a `.zip` file and Safari will unpack it (by default) which will make it not functional. It has to be downloaded as `.zip` file and installed using `.zip` file, not with `.jar` file inside zip/unpacked folder.
* At moment of writing latest release is `2.3.8` and latest snapshot is `2.3.9-SNAPSHOT`.
* `2.3.10-SNAPSHOT` is experimental as contains many introduced changes in compier/api/bindings which were not deeply tested. While `2.3.9-SNAPSHOT` version is being kept for possible hotfixes.
* `2.3.10-SNAPSHOT` is build from [jdk12 branch](https://github.com/MobiVM/robovm/tree/jdk12) and it names might be confusing. It doesn't deliver JDK12 functionality to RoboVM but was opened to allow RoboVM compiler be running on JDK9+. As result it hosts 2.3.10 version.

## What's changed
### Source tree: [migrating to jdk12]({{ site.baseurl }}{% post_url 2019-08-06-migrating-jdk-12 %})
- maven project structure reformated;
- removed sonatype root parent pom;
- improved dependency management (moved to root pom);
- plugin versions/dependencies updated to recent where possible;
- unit tests fixed to run on jdk9+/openjdk.

### Bindings
- all classes in CocoaTouch library is compilable now. Hooray!
- [iOS13 bindings]({{ site.baseurl }}{% post_url 2019-09-18-ios-13-binding %}), [pr406](https://github.com/MobiVM/robovm/pull/406);
- Network framework bindings re-worked: [PostFix #6]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %}), [pr439](https://github.com/MobiVM/robovm/pull/439);
- iOS13.2 bindings: [PostFix #7]({{ site.baseurl }}{% post_url 2019-12-23-cocoa13-postfix7-ios-13-2-bindings %}), [pr441](https://github.com/MobiVM/robovm/pull/441);
- glkit: ported missing functions (static inline now): [PostFix #10]({{ site.baseurl }}{% post_url 2020-01-03-cocoa13-postfix10-glkit-adding-missing-inline-fn %}), [pr441(ios13.2)](https://github.com/MobiVM/robovm/pull/441);

### Compiler changes (fixes and new tricks)
- Generic class arguments and @Block parameters: [PostFix #1]({{ site.baseurl }}{% post_url 2019-10-19-cocoa13-postfix1-generic-blocks %}), [pr419](https://github.com/MobiVM/robovm/pull/419)
- Support for non-static @Bridge method in enums classes: [PostFix #2]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %}), [pr420](https://github.com/MobiVM/robovm/pull/420)
- Support for @Block member in structs: [PostFix #3]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %}), [pr421](https://github.com/MobiVM/robovm/pull/421)
- [fixed]Compilation failed on @Bridge annotate covariant return synthetic method: [PostFix #4]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %}), [pr422](https://github.com/MobiVM/robovm/pull/422)
- Support for Struct.offsetOf in structs: [PostFix #5]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %}), [pr431](https://github.com/MobiVM/robovm/pull/431)
- workaround for missing objc classes(ObjCClassNotFoundException), [PostFix #8]({{ site.baseurl }}{% post_url 2019-12-24-cocoa13-postfix8-fix-for-missing-classes %}), [pr442](https://github.com/MobiVM/robovm/pull/442)
- *Hooray!* experimental and formal bitcode support, [PostFix #9]({{ site.baseurl }}{% post_url 2019-12-26-cocoa13-postfix9-formal-bitcode-support %}), [pr443](https://github.com/MobiVM/robovm/pull/442)
- [fix] `error code -34018` when using Security API on simulator: [pr447](https://github.com/MobiVM/robovm/pull/447), [post]({{ site.baseurl }}{% post_url 2020-01-05-simulator-enabling-security-framework-err-34018-fix %})

### Idea plugin
- fixed annoying Android gradle faced [#242](https://github.com/MobiVM/robovm/issues/242), [post]({{ site.baseurl }}{% post_url 2019-08-15-android-studio-workaround %});
- plugin is now built with gradle script, see [migrating to jdk12]({{ site.baseurl }}{% post_url 2019-08-06-migrating-jdk-12 %});
- code was cleaned up, upgraded to Java8 code level, and deprecated API usage were removed where possible to support Idea 2019.3,  "no Xcode dialog" is not blocking anymore: [pr434](https://github.com/MobiVM/robovm/pull/434);
- disabled `generate separate IDEA module per source set`: [pr449](https://github.com/MobiVM/robovm/pull/449), [post]({{ site.baseurl }}{% post_url 2020-01-09-idea-disabling-separate-module-per-source-set %});

Happy coding!
Please report any issue to [tracker](https://github.com/MobiVM/robovm/issues/new).
