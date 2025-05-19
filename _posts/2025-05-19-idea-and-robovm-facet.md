---
layout: post
title: 'Intellij Idea maintenance + RoboVM facet'
tags: ['idea-plugin']
---
![]({{ "/assets/2025/05/19/robovm-facet.png"}})

## major goals:
- removal of outdated not available Components;
- fix for annoying UI freeze in projects with many modules;
<!-- more -->

```
Freeze in EDT for 82 seconds
Sampled time: 33700ms, sampling rate: 100ms, GC time: 3016ms (3%), Class loading: 0%
com.intellij.diagnostic.Freeze
    at com.intellij.openapi.vfs.newvfs.VfsImplUtil.findFileByPath(VfsImplUtil.java:56)
    ...
    at com.intellij.openapi.roots.OrderEnumerator.getPathsList(OrderEnumerator.java:158)
    at org.robovm.idea.RoboVmPlugin.isRoboVmModule(RoboVmPlugin.java:463)
```

## Details 1. `ApplicationComponent`/`ProjectComponent`/`ModuleComponent` are declared outdated and some API are already removed.

### `RoboVmApplicationComponent`
Was responsible for SDK file unpacking, JDK selection dialog and XCode missing. Replaced with:
- `RoboVmAppSetupService` -- on demand service that show XCode related dialog and does debugger setup;
- `RoboVmSdkUnpackService` -- on demand service that unpacks SDK files;
- `RoboVmProjectStartupActivity` -- triggers once project is loaded and calls `RoboVmAppSetupService` to handle Xcode installation and debugger setup;

### `RoboVmProjectComponent`
Was triggered once project is loaded and was calling `RoboVmPlugin.initializeProject(project)`, replaced with `RoboVmProjectStartupActivity`.


### `RoboVmSdkUpdateComponent`
Was observing for each module content change notifications to detect if module is RoboVM one and setup RoboVM SDK to it.

## Details 2. RoboVM facet
Facets are using for attaching specific configuration to modules. Quite sarcastic as it was quite annoying with [android facet]({{ site.baseurl }}{% post_url 2018-05-04-idea-fixing-android-gradle-facet3 %}) that was breaking RoboVM/Java projects and now RoboVM facet is introduced.   
It is being used to `mark` modules as RoboVM ones during project import based on gradle/maven build system only in cases where RoboVM plugin is used.
That is a true way to mark and later recognizes modules as RoboVM ones.  
It fixes previous logic for recognizing modules: each module's content root was scanned for `robovm.xml` file and only in this case it was assumed as RoboVM one.
Such scan was happening:
- on every need to find robovm module: when showing modules in `Create IPA`/`Run`/`Open Interface Builder` actions. In case of big projects this cause UI freeze and often made Idea to hung;
- on each content root change (`RoboVmSdkUpdateComponent`) during project import and so on;

More over such logic produced false detection results and as result if there was `robovm.xml` files located in directory SDK of such module was forcly replaced with RoboVM one.

### Gradle build system
`RoboVMGradleProjectResolver` recognizes RoboVM plugin activated in `build.gradle` and attaches own data to DataNode of imported node. Later `RoboVmModuleDataService` recognize this data and configure module with Facet/SDK.

### Maven build system
`RoboVmMavenImporter` implements `FacetImporter` -- an outdated but still in use class that just automatically creates facet for specified plugin group/artifact id. All that to be done is to attach facet/configure SDK.

### Intellij Idea managed project
Project created from template -- automatically being attached with facet/sdk. In other cases its possible to attach RoboVM facet to module in `Project Structure` dialogue.

## Details 3. Other changes
- JDK selection dialog is removed -- Android Studio/Intellij Idea has shipped JDK and which is being created automatically (probably this might be required to be adjusted);
- RoboVM SDK is not JDK anymore. As result it can't be created manually and can't be specified as Gradle SDK. We had issues with this as RoboVM SDK often was proposed as one that has full set of Java tools;
- Only one RoboVM SDK can exist. All copies are being dropped. There is no need to keep previous/incompatible versions.
- Part of code was converted from Java to Kotlin.

### Code 
Changes proposed as [PR808](https://github.com/MobiVM/robovm/pull/808)

