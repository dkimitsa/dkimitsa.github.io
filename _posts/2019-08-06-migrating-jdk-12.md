---
layout: post
title: 'RoboVM updated to run on JDK12 host'
tags: [jdk12, maven]
---

![]({{ "/assets/2019/8/7/java8-eol.png"}})

Changes propose as [PR400](https://github.com/MobiVM/robovm/pull/400)

As Java8 is end of support and requires additional registration to be downloaded its time to support recent versions of Java. Switching to it is not simple case as it requires complex set of changes to sources, tests and dependencies. 

## Code and unit tests update 
First thing that was failing when trying to run on JDK12 are compiler's tests. Followings were the issues: 
<!-- more -->
* tests were manipulating `user.dir` to get different base dir, but this behaviour changed in JDK9+ (changing `user.dir` has no effect any more);
* in JDK9+ File.getAbsolutePath() turns not absolute path to absolute using `user.dir` which results in broken logic that finds relative pathes in Config.java;
* in JDK9+ there is no more `rt.jar` as all runtime is split to modules. As result tests were failing to resolve basic classe like `java.lang.Object`. Both RoboVM and Soot were adapted; 
* issues with [ByteBuffer in JDK9+](https://www.reddit.com/r/androiddev/comments/ah194v/javaniobytebuffer_java_9_compiler_and_android/) was not fixed with casting to `buffer` but with enforcing `-release 8` parameter to java compiler;

## Soot update 
Soot [was updated](https://github.com/dkimitsa/soot/commit/6fa8247bbc2244be2c18b269b22168ffadfaa6da) to support runtime classes loading from modules. Changes were picked from original Soot repo. 

## Maven build script update
Maven scripts were reviewed and following changes were applied:
* all dependency management were moved to root pom file;
* root pom file has no parent anymore as oss-parent is few year outdated and enforces outdated dependencies;
* all dependencies and plugins are updated to recent versions;
* compiler plugin configured to enforce '-release 8'
* RoboVMs runtime/cocoa touch/objc libraries are now being compiled without JDKs boot classes, so there is no more conflicts and allows recent java to be used to compile them;
* simple rule to generate empty JavaDoc and Sources packages: now it is enough to have `src/empty` folder which will enable the rule and disables javadoc and sources maven plugins; 

## Idea plugin 
Package build migraged to gradle as it suffers due missing `-extdir` argument in JDK9+ which makes dependency management compilicated. Maven configuration left for development purposes only and greatly simplified. Also added `ij.pluginDescriptor` argument to pom file that removes the need to configure `META-INF/plugin.xml` path. Refer README.md for details.

## Eclipse plugin 
Can be build with JDK11 only, as JDK12 is not supported yet by Eclipse Tycho.  
Eclipse plugin was not checked if it is working.

