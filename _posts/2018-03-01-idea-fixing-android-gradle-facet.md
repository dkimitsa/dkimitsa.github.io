---
layout: post
title: "Intellij Idea/Android Studio: fixing annoying 'android-gradle' facet"
tags: ["dirty hack", idea, gradle ]
---
**This post is outdated**. Please follow to [post]({{ site.baseurl }}{% post_url 2018-03-06-idea-fixing-android-gradle-facet2 %}).

Last year AS/Idea become real pain in case Java project is mixed with Android one. This results of "android-gradle" facet and project type to be created. And these project type doesn't play nice with RoboVM/Java projects. Nothing is working.  
There is already several issues about this in the field:  
* [IDEA-122904 Android-Gradle facet added to non-Android modules upon import](https://youtrack.jetbrains.com/issue/IDEA-122904)
* [RoboVM: Kotlin classes in library module not being compiled](https://github.com/MobiVM/robovm/issues/264)
* [RoboVM: Unable to start RoboVM iOS in IntelliJ due to NullPointerException](https://github.com/MobiVM/robovm/issues/242)  
And so on. The bug is in downstream dependency -- [github/Android plugin](https://github.com/JetBrains/android) for Idea.

It is almost year as JetBrains fixes it. So there is a hack which allows to run it:   
<!-- more -->
**DISCLAIMER**: Do not know if this way is correct and it would have side effects I don't know about but at least it allows RoboVM to run.   
(the fix is for `173.4548` version of Idea)

**This patch is outdated**. Please follow to [post]({{ site.baseurl }}{% post_url 2018-03-06-idea-fixing-android-gradle-facet2 %}).

```patch
Index: android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
--- android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java	(revision 099db0922f5c4680779ca7946925dd58b825f378)
+++ android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java	(date 1519905777000)
@@ -224,17 +224,7 @@
       return;
     }

-    BuildScriptClasspathModel buildScriptModel = resolverCtx.getExtraProject(BuildScriptClasspathModel.class);
-    String gradleVersion = buildScriptModel != null ? buildScriptModel.getGradleVersion() : null;
-
-    File buildFilePath = buildScript.getSourceFile();
-    GradleModuleModel gradleModuleModel = new GradleModuleModel(moduleName, gradleProject, buildFilePath, gradleVersion);
-    ideModule.createChild(GRADLE_MODULE_MODEL, gradleModuleModel);
-
-    if (nativeAndroidProject == null && (androidProject == null || androidProjectWithoutVariants)) {
-      // This is a Java lib module.
-      createJavaProject(gradleModule, ideModule, androidProjectWithoutVariants);
-    }
+    nextResolver.populateModuleContentRoots(gradleModule, ideModule);
   }

   private void createJavaProject(@NotNull IdeaModule gradleModule,
```

## What in the fix
The fix changes the logic to following:
- if it is not Android module
- and if it is not AndroidNDK module
- and if it is not Root module (with settings.gradle)
- just pass control to Next resolver instead of creating `android-gradle` project.


## How to apply fix
Checkout Android plugin, make changes and replace it in Idea installation folder (long way). Or short way:  

### Jar hacking
Lets just compile single `AndroidGradleProjectResolver.java` and replace this class file inside `android.jar`:

1. Create empty Java project in Idea (just pure java, without gradle and other stuff);
2. Get `AndroidGradleProjectResolver.java` from [github/Android plugin](https://github.com/JetBrains/android) for Idea. Make sure to pick up version that matches Idea you are running;
3. Copy  `AndroidGradleProjectResolver.java` to src folder and fix package to expected;
4. Add missing dependencies from Idea installation, following jar libs is enough:
    * `/Applications/IntelliJ IDEA CE.app/Contents/lib`
    * `/Applications/IntelliJ IDEA CE.app/Contents/plugins/android/lib`
    * `/Applications/IntelliJ IDEA CE.app/Contents/plugins/gradle/lib`
5. Compile it
6. pickup `AndroidGradleProjectResolver.class` from `out` folder and replace with it corresponding file in `/Applications/IntelliJ IDEA CE.app/Contents/plugins/android/lib/android.jar`
