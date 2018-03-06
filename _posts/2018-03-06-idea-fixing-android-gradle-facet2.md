---
layout: post
title: "Intellij Idea/Android Studio: fixing annoying 'android-gradle' facet and dependencies"
tags: ["dirty hack", idea, gradle ]
---
[Previous post]({{ site.baseurl }}{% post_url 2018-03-01-idea-fixing-android-gradle-facet %}) described how to remove `android-gradle` facet but as well with it there was module dependency resolution broken. It is time to fix it as well.  
## What was wrong
It was missed to handle properly `populateModuleDependencies` as result all modules were not populated with dependencies. As modules were moved out of Android gradle structures this tast shall be also forwarded to nextResolver.

<!-- more -->
## Important -- clean up project
If project was already used with original `android.jar` there is a lot of data stuck in Idea files. To get clean result following is recomended:
- close project
- delete all `iml` files
- delete `.idea` folder -- **warning** some settings might be lost;
- open project again and all these will be regenerated

## Important2 -- turn off "Create separate module per source set"
There is a bug when this option is on and module dependency is replaced to jar dependency. Having it "off" solves issue.

## how to get patched android.jar
Please read [previous post]({{ site.baseurl }}{% post_url 2018-03-01-idea-fixing-android-gradle-facet %}) it describes simple jar hacking method (class file replacement in .jar) without need to build entire android project.

## The patch
```patch
Index: android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
--- android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java	(revision 099db0922f5c4680779ca7946925dd58b825f378)
+++ android/src/com/android/tools/idea/gradle/project/sync/idea/AndroidGradleProjectResolver.java	(date 1520342730000)
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
@@ -248,7 +238,7 @@

   @Override
   public void populateModuleCompileOutputSettings(@NotNull IdeaModule gradleModule, @NotNull DataNode<ModuleData> ideModule) {
-    if (!isAndroidGradleProject()) {
+    if (!isAndroidGradleProject() || !isAndroidModule(gradleModule, ideModule)) {
       nextResolver.populateModuleCompileOutputSettings(gradleModule, ideModule);
     }
   }
@@ -257,12 +247,26 @@
   public void populateModuleDependencies(@NotNull IdeaModule gradleModule,
                                          @NotNull DataNode<ModuleData> ideModule,
                                          @NotNull DataNode<ProjectData> ideProject) {
-    if (!isAndroidGradleProject()) {
+    if (!isAndroidGradleProject() || !isAndroidModule(gradleModule, ideModule)) {
       // For plain Java projects (non-Gradle) we let the framework populate dependencies
       nextResolver.populateModuleDependencies(gradleModule, ideModule, ideProject);
     }
   }

+  private boolean isAndroidModule(@NotNull IdeaModule gradleModule, @NotNull DataNode<ModuleData> ideModule) {
+    AndroidProject androidProject = resolverCtx.getExtraProject(gradleModule, AndroidProject.class);
+    if (androidProject != null)
+      return true;
+
+    NativeAndroidProject nativeAndroidProject = resolverCtx.getExtraProject(gradleModule, NativeAndroidProject.class);
+    if (nativeAndroidProject != null)
+      return true;
+
+    File moduleRootDirPath = new File(toSystemDependentName(ideModule.getData().getLinkedExternalProjectPath()));
+    File gradleSettingsFile = new File(moduleRootDirPath, FN_SETTINGS_GRADLE);
+    return gradleSettingsFile.isFile();
+  }
+
   // Indicates it is an "Android" project if at least one module has an AndroidProject.
   private boolean isAndroidGradleProject() {
     Boolean isAndroidGradleProject = resolverCtx.getUserData(IS_ANDROID_PROJECT_KEY);
```
