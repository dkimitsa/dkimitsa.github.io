---
layout: post
title: "Intellij Idea/Android Studio: never ending battle with annoying 'android-gradle'"
tags: ["dirty hack", idea, gradle ]
---
[UPDATE: possible solution delivered]({{ site.baseurl }}{% post_url 2019-08-15-android-studio-workaround %})   
[Previous post on this topic]({{ site.baseurl }}{% post_url 2018-03-06-idea-fixing-android-gradle-facet2 %}) described how to hack `android.jar`.   
That approach has own flaws such as it is not working properly with recent Android Studio. Now there is another way to fight the issue -- remove `android-gradle` and `java-gradle` in automatic way.  
<!-- more -->
## Way to fix
To have it done automatically I just wrote simple plugin with only duty to remove these annoying facets.  
Source code is available at my [codesnippets](https://github.com/dkimitsa/codesnippets/tree/master/AndroidGradleFacetFix) github repo.  
Compiled plugin can be downloaded from [here](https://github.com/dkimitsa/codesnippets/raw/master/AndroidGradleFacetFix/lib/AndroidGradleFacetFix.jar).  
Plugin has 'Fix for Android Gradle Facet' name in the list and can be removed any minute.

## What it does?
1. Registers as `ProjectComponent` to get notified once project is created;
2. For project it subscribes for `Project wide facet change` notifications;
```java
public AndroidFacetRemoverProjectComponent(Project p) {
    myProject = p;
    ProjectWideFacetListenersRegistry.getInstance(myProject).registerListener(new ProjectWideFacetAdapter<Facet>(){
        @Override
        public void facetAdded(Facet facet) {
            if (facet.getTypeId().equals(GradleFacet.getFacetTypeId()))
                removeExtraFacets();
        }
    });
}
```
3. If there is `android-gradle` added it triggers facet cleanup:
```java
private void removeExtraFacets() {
   ApplicationManager.getApplication().runWriteAction(() -> {
       for (Module module : ModuleManager.getInstance(myProject).getModules()) {
           if (AndroidFacet.getInstance(module) == null && GradleFacet.getInstance(module) != null) {
               ModifiableFacetModel model = FacetManager.getInstance(module).createModifiableModel();
               model.removeFacet(GradleFacet.getInstance(module));
               facetsRemoved = GradleFacet.getFacetId();
               if (JavaFacet.getInstance(module) != null) {
                   model.removeFacet(JavaFacet.getInstance(module));
               }
               model.commit();
           }
       }
   });
}
```


## IntelliJ Idea
Just install the plugin and it will be removing the facets. It will output to `Event-log` messages about modules being cleaned-up:
>1:15 PM	Facets removed: android-gradle from module ios  
>1:15 PM	Facets removed: android-gradle from module core

## Google Android Studio
It is almost not usable with Java or RoboVM projects as it is highly focused around android. Simple `Project structure` looks horrible in it and is not usable (no java or robovm modules etc):
![]({{ "/assets/2018/05/04/as-unusable-project-structure.png" | absolute_url}})  

Solution is to run `Android Studio` in `IntelliJ Idea` mode, simple run it without specifying platform prefix with shell script similar to this:
```bash
export CS="${JAVA_HOME}/lib/tools.jar"
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/log4j.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/jdom.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/trove4j.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/openapi.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/util.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/extensions.jar"  
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/bootstrap.jar"
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/idea_rt.jar"
export CS="${CS}:/Applications/Android Studio.app/Contents/lib/idea.jar"
${JAVA_HOME}/bin/java -classpath "${CS}" com.intellij.idea.Main
```

After this plugin can be installed and AS will act mostly as Idea but will keep all android related stuff.
