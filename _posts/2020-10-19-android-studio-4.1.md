---
layout: post
title: 'RoboVM and AndroidStudio 4.1'
tags: [idea, workaround, tutorial]
---
Dealing with Android Studio always tricky, v4.1 brings another set of issues to solve. 

## Installing the plugin
`2.3.10` is available in JetBrains marketplace and can be installed directly from `plugin` settings dialog. But due wrong configuration it is not visible in Android studio. There are few options for Android studio:  
### Install manually
Download .zip file from [MobiVM site](http://robovm.mobidevelop.com/downloads/releases/idea/) and use `Install Pluging From Disk...` option.    
Important: Safari will automatically unpack .zip file, it should be downloaded without unpacking and installed as .zip!

### Install from custom plugin repository
These custom repositories point to files located at [MobiVM site](http://robovm.mobidevelop.com/downloads/releases/idea/).  
In settings dialog: 
* select `Manage Plugin Repositories`;
* add `https://dkimitsa.github.io/assets/mobivm-release.xml` for installing the latest release;
* or add `https://dkimitsa.github.io/assets/mobivm-snapshot.xml` for installing the latest snapshot;

Then switch to `Market place` search for `MobiVM`;  


## Not working RoboVM/New Project(Module) menu 
<!-- more -->
Android Studio doesn't have new Project/Module menu option and RoboVM delivers workaround. In AS 4.1 it stopped working.  
Failure caused by exception in `kotlin-plugin v1.4`:  
```
com.intellij.openapi.extensions.ExtensionInstantiationException: java.lang.ClassNotFoundException: org.jetbrains.kotlin.tools.projectWizard.wizard.NewProjectWizardModuleBuilder PluginClassLoader[org.jetbrains.kotlin, 1.4.10-release-Studio4.1-1] com.intellij.ide.plugins.cl.PluginClassLoader@64d3a9e5 [Plugin org.jetbrains.kotlin]
    at com.intellij.openapi.extensions.AbstractExtensionPointBean.findExtensionClass(AbstractExtensionPointBean.java:51)
    at com.intellij.openapi.extensions.AbstractExtensionPointBean.instantiateClass(AbstractExtensionPointBean.java:92)
    at com.intellij.ide.util.projectWizard.ModuleBuilderFactory.createBuilder(ModuleBuilderFactory.java:31)
```

Root case -- bug in kotlin-plugin bundled with AS 4.1:
- `NewProjectWizardModuleBuilder` was moved to standalone `.jar` that is not deployed with AS anymore;
- But this class is still registered with `extensions/ide.xml`.

Issue was reported to Jetbrains: [KT-42779](https://youtrack.jetbrains.com/issue/KT-42779).

Workaround: remove registration from the plugin, simple script that unpacks-fixes-packs back:  
```
unzip ~/Library/Application\ Support/Google/AndroidStudio4.1/plugins/Kotlin/lib/kotlin-plugin.jar META-INF/extensions/ide.xml -d unpack
cat unpack/META-INF/extensions/ide.xml | sed 's|<moduleBuilder builderClass="org.jetbrains.kotlin.tools.projectWizard.wizard.NewProjectWizardModuleBuilder"/>||g' > unpack/META-INF/extensions/ide.xml.2
mv unpack/META-INF/extensions/ide.xml.2 unpack/META-INF/extensions/ide.xml
cd unpack; zip -r ~/Library/Application\ Support/Google/AndroidStudio4.1/plugins/Kotlin/lib/kotlin-plugin.jar *
```

## Tutorial: adding iOS module to Android app
Not a bug but been often asked: 
1. Add new module using menu: `RoboVM -> New -> New Module...`;
2. Module type: `RoboVM -> RoboVM iOS App without storyboard`;
3. Make sure that `Content root` and `Module file location` set to differt from Android module location (and will not override it) -- e.g. `ios`;

After this gradle sync will fails with message:  
> Project directory 'ios' is not part of the build defined by settings file 'settings.gradle'
> If this is an unrelated build, it must have its own settings file.

Add it to `settings.gradle` as described in [Gradle guide](https://docs.gradle.org/current/userguide/multi_project_builds.html):   
> include ':app', ':ios'

Next sync will fail with the message:  
> Task 'wrapper' not found in project ':ios'.

Adding the task to `ios/build.gradle` to fix:  
```
task wrapper(type: Wrapper) {
    distributionType = Wrapper.DistributionType.ALL
}
```

Next sync will fail with the message:   
> Task 'prepareKotlinBuildScriptModel' not found in project ':ios'.

Root case of it -- bug in [Gradle Plugin 4.1.0 and Gradle 6.5](https://github.com/gradle/gradle/issues/14889).  
Workaround is possible as described in issue above with reverting to working versions:
Update `build.gradle` with `4.0.1`:   
```
dependencies {
        classpath "com.android.tools.build:gradle:4.0.1"
}
```

Update `gradle/wrapper/gradle-wrapper.properties` with `6.1.1`:
```
distributionUrl=https\://services.gradle.org/distributions/gradle-6.1.1-all.zip
```

Hooray! Sync is successful, ios module can be built and running!
