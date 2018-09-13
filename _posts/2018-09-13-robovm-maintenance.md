---
layout: post
title: 'RoboVM maintenance: keeping up to date'
tags: [idea, xcode, gradle]
---
Today maintenance [PR326](https://github.com/MobiVM/robovm/pull/326) delivers following:
* fixes gradle's ` Warning: org.robovm.apple.uikit.UIWindow is a phantom class!` [issue](https://gitter.im/MobiVM/robovm?at=5b98e2c4f3c26b08f6664e07) when using v4 and `implementation` as dependency command. This was happening due using outdated configuration name in dependency path resolution:  

```java
// configure the runtime classpath
Set<File> classpathEntries = project.getConfigurations().getByName("runtime").getFiles();
```

while  
```java
 // The name of the "runtime" configuration. This configuration is deprecated and doesn't represent a correct view of
 // the runtime dependencies of a component.
```

* update to Idea 2018.2.3: fixes broken Idea plugin build due to gradle dependencies needs to be specified
* update to Xcode10: sdk 10.14 and removed x86 mac target as deprecated and breaks build process
