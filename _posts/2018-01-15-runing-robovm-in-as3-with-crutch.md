---
layout: post
title: 'AndroidStudio3: using RoboVM/gradle project with a crutch'
tags: ['dirty hack', 'idea-plugin', 'gradle']
---

There is old living bug that RoboVM doesn't work nice in [Android Studio 3.0](https://github.com/MobiVM/robovm/issues/193)
And it still doesn't work nice, here few workarounds that will allow to start project.

<!-- more -->

1. With [PR253](https://github.com/MobiVM/robovm/pull/253) there is a menu that allows to bring "create new project" dialog back.
![]({{ "/assets/2018/01/12/robovm-idea-new-menu.png" | absolute_url}})

2. Also same [PR253](https://github.com/MobiVM/robovm/pull/253) fixes crashes that appeared during gradle project importing due different `ProjectDataManager` are used in `Idea 2017.3` and `Android Studio 3` (last one is used deprecated and recent class is not present in AS3)

3. There is an [epic bug](https://youtrack.jetbrains.com/issue/IDEA-184950) discovered in Idea/AS which costed me half of the day. The bug itself says that "gradle version is unsupported", the workaround would be a try to use different (but supported) version (or file manipulation but I will not cover it here as not realible). Check [link](https://youtrack.jetbrains.com/issue/IDEA-184950) for details;
![]({{ "/assets/2018/01/15/gradle-version-unsupported.png" | absolute_url}})

4. Once gradle project is being imported and "gradle version is unsupported" message appear either try different version of gradle or hit ok and AS will add gradle wrapper to project

5. Most likely during import following error will be received. It should be ignored for now as Android Studio forces 'android-gradle' builder even for pure "gradle-java" project. It is not clear for now how to fight it.
> Unsupported Modules Detected: Compilation is not supported for following modules: robo. Unfortunately you can't have non-Gradle Java modules and Android-Gradle modules in one project.

6. if project doesn't compile with error message:  
`[ERROR] Couldn't compile app org.robovm.compiler.CompilerException: Main class com.mycompany.myapp.Main not found`  
try following:
   - open gradle panel;
   - re-import gradle project;
   - open module settings and remove `android-gradle` and keep only `Java-gradle`;
   - if gradle project is refreshed -- remove `android-gradle` again
   ![]({{ "/assets/2018/01/15/gradle-use-java-gradle.png" | absolute_url}})

7. instead of 6. it is possible just trigger project build from `Build-Make project` menu;

*Bottom line*
- `Intellij Idea` and `Android Studio` already have different code base, this makes hard to support;
- `RoboVM Idea` plugin is quite old and requires refactoring especially gradle import part, possible this will fix issues in AS.
- Unpleasant thing here is that it is not possible to debug plugin in Android Studio, this makes process time consuming and difficult.
- for now use `Intellij Idea` if possible
