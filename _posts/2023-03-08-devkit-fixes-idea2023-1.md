---
layout: post
title: 'Idea 2023.1: DevKit and fixes'
tags: [idea]
---

DevKit is legacy tool to develop Intellij Idea plugins. It is being used for development of RoboVM compiler and Idea plugin as have benefits of seamless debugging and code hotswap.  
Sadly starting from Idea 2022.3 it stopped to Run/Debug projects with following message:
> Error: Could not find or load main class com.intellij.idea.Main 
<!-- more -->
There are few reports already [IDEA-314787](https://youtrack.jetbrains.com/issue/IDEA-314787), [IDEA-314787](https://youtrack.jetbrains.com/issue/IDEA-314787) but fix not seems to be delivering at high priority. At least recent Idea 2023.1 still has this issue. 

## Root case 
DevKit starts another Idea instance and specifies following `jar` files as class path ([PluginRunConfiguration.java](https://github.com/JetBrains/intellij-community/blob/09967377a46215dab4335fe23bdbf0e39f4e3de2/plugins/devkit/devkit-core/src/run/PluginRunConfiguration.java#L246)):
```java
final List<String> jars = List.of(
      // log4j, jdom and trove4j needed for running on branch 202 and older
      "log4j.jar", "jdom.jar", "trove4j.jar",
      "openapi.jar", "util.jar", "util_rt.jar", "bootstrap.jar", "idea_rt.jar", "idea.jar",
      "3rd-party-rt.jar", "jna.jar");
```

Since Idea 2022.2 `com.intellij.idea.Main` is not available in `util.jar` anymore. 

## The fix 
Adding missing class path entries to VM params of run configuration doesn't work (and messy due spaces in path) as only last entry of `-cp` is used by java.  
And fasted way is to patch `PluginRunConfiguration.java` of `plugins/devkit/lib/devkit.jar` itself by adding proper list of required jars.
The list of required class path to start Idea is located in `/Applications/IntelliJ IDEA 2023.1 CE EAP.app/Contents/Info.plist`, JVMOptions/ClassPath. This results in following code:
```java
final List<String> jars = List.of(
        "app.jar", "3rd-party-rt.jar", "util.jar", "util_rt.jar", "util-8.jar",
        "jps-model.jar", "stats.jar", "protobuf.jar", "external-system-rt.jar", "intellij-test-discovery.jar",
        "forms_rt.jar", "rd.jar", "externalProcess-rt.jar", "annotations-java5.jar",
        "annotations.jar", "byte-buddy-agent.jar", "error-prone-annotations.jar",
        "groovy.jar", "idea_rt.jar", "intellij-coverage-agent-1.0.706.jar", "jsch-agent.jar",
        "junit.jar", "junit4.jar", "ant/lib/ant.jar");
```

Simplest way to patch it:
- create dummy DevKit theme project with Idea platform SDK;
- add `plugins/devkit/lib/devkit.jar` as dependency to project;
- add [PluginRunConfiguration.java](https://github.com/JetBrains/intellij-community/blob/09967377a46215dab4335fe23bdbf0e39f4e3de2/plugins/devkit/devkit-core/src/run/PluginRunConfiguration.java#L246)
- make changes and compile
- pick compiled class files and replace ones in `plugins/devkit/lib/devkit.jar`

## Bonus: using Maven based projects for DevKit
For Idea plugin/RoboVM Compiler development maven presentation of project was used. It allows seamless debugging and code hotswap. 
To resolve Maven Java project into DevKit project [Intellij plugin development with Maven](https://plugins.jetbrains.com/plugin/7127-intellij-plugin-development-with-maven) was used. Sadly it is not supported and not working anymore in recent Idea. Last release was in 2016.

It is really nice to have this functionality back so time was invested and another hacky plugin was coded that does same in simple hacky way:
- once Maven Java module is created its type is switched to from `JAVA_MODULE` to `PLUGIN_MODULE`
- old way is not working anymore as Maven unconditionally applies `JAVA_MODULE` to created modules. 

Code is simple as bellow: 
```java
public class DevKitPluginMavenModuleImporter extends MavenImporter {
    @Override
    public boolean isApplicable(MavenProject mavenProject) {
      return Boolean.parseBoolean(mavenProject.getProperties().getProperty("ij.plugin"));
    }

    @Override
    public void process(@NotNull IdeModifiableModelsProvider modifiableModelsProvider, @NotNull Module module,
                        @NotNull MavenRootModelAdapter rootModel, @NotNull MavenProjectsTree mavenModel,
                        @NotNull MavenProject mavenProject, @NotNull MavenProjectChanges changes,
                        @NotNull Map<MavenProject, String> mavenProjectToModuleName,
                        @NotNull List<MavenProjectsProcessorTask> postTasks) {
        // hack the module type
        module.setModuleType(PluginModuleType.ID);
    }
}
```

Plugin source code is pushed to repository [dkimitsa/support-maven-devkit-plugins](https://github.com/dkimitsa/support-maven-devkit-plugins). 


