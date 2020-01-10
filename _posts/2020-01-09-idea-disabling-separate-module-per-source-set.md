---
layout: post
title: "Idea: disabling `generate separate IDEA module per source set`"
tags: [fix, idea, gradle]
---

Idea 2019.2 deprecated `generate separate IDEA module per source set` option as mentioned in their [blog post, reworked Gradle settings dialog](https://blog.jetbrains.com/idea/2019/05/intellij-idea-starts-the-2019-2-early-access-program/). For RoboVM this option is important as it is required to be disabled. Having it enabled it will produce three modules once gradle file is imported:
```
module (content root: ./ )
    module-main (content root: ./src/main)
    module-test (content root: ./src/test)
```

## Root case
Its a problem for Idea plugin to find out the module eligible to run. As one of criteria is presence of `robovm.xml` in content root of module. Example above shows that module with code is `module-main` but its content root points to source folder not module root folder itself (and there is no robovm.xml). Group module `module` contains valid content root with `robovm-xml` but no source root. As result run/debug dialog shows no valid module to run/debug.

## Workaround
<!-- more -->
As mentioned in several issues on Jetbrain's tracker (like [IDEA-222172](https://youtrack.jetbrains.com/issue/IDEA-222172)) it is possible to disable option by manually editing .idea/gradle.xml file and adding `resolveModulePerSourceSet` key as in example bellow with following reimporting of project:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="GradleSettings">
    <option name="linkedExternalProjectsSettings">
      <GradleProjectSettings>
        ....
        <option name="useQualifiedModuleNames" value="false" />
        <!-- workaround -->
        <option name="resolveModulePerSourceSet" value="false" />
      </GradleProjectSettings>
    </option>
  </component>
</project>
```

## the fix
The fix is to disable option while Idea importing the project for RoboVM module. This is done by registering and implementing Gradle Project Resolver in Idea plugin. It does following:
### for each detected RoboVM module disables separate module per source set:
```java
public DataNode<ModuleData> createModule(@NotNull IdeaModule gradleModule, @NotNull DataNode<ProjectData> projectDataNode) {
    if (isRoboVMModule(gradleModule))
        resolverCtx.getSettings().setResolveModulePerSourceSet(false);
    return super.createModule(gradleModule, projectDataNode);
}
```

### detecting RoboVM module
To detect RoboVM module [gradle tooling api](https://docs.gradle.org/current/userguide/third_party_integration.html#embedding) is used. To have it possible RoboVM gradle plugin was extended to provide simple tooling model that can be accessible from Idea:
```java
public interface RoboVMGradleModel {
    String getVersion();
}
public static class DefaultRoboVMGradleModel implements RoboVMGradleModel, Serializable {
    @Override
    public String getVersion() {
        return RoboVMPlugin.getRoboVMVersion();
    }
}
```

And module is RoboVM one if its build.gradle applies RoboVM plugin and support tooling:
```java
private boolean isRoboVMModule(IdeaModule gradleModule) {
    return resolverCtx.getExtraProject(gradleModule, RoboVMGradleModel.class) != null;
}
```

*IMPORTANT*: solution will work ONLY if build.gradle uses RoboVM gradle plugin that supports tooling, e.g. version 2.3.10+.
Code was delivered as [PR449](https://github.com/MobiVM/robovm/pull/449)


