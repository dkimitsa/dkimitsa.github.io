---
layout: post
title: 'Workaround for AndroidStudio/Fix242/Maintaince'
tags: [idea, fix]
---
This post continue series of [Android facet]({{ site.baseurl }}{% post_url 2018-05-04-idea-fixing-android-gradle-facet3 %}) related posts and delivers a workaround for [issue #242 Unable to start RoboVM iOS in IntelliJ due to NullPointerException](https://github.com/MobiVM/robovm/issues/242).  
Fix is delivered as [PR401](https://github.com/MobiVM/robovm/pull/401).  

## Root cause 
It happens as long as project gets Android module (which attaches AndroidFacet) to all modules and Idea loses compilation control as Android plugin takes over it. RoboVM Idea plugin registers as compilation step to process compiled java class files into native objects. Buts as this not happens as Android plugin respects not more than gradle rules (e.g. ignores Idea's compilations steps) native parts is not get generated and plugin crashes with NPE. 
Also it will not work if build and run is configured to be done with gradle and not with Idea. Its being controlled by these options:   
![]({{ "/assets/2019/8/15/idea-gradle-options.png" | absolute_url}})

## The fix
<!-- more -->
Proper one: assume that is best way is to integrate into gradle build process similar as Fabric does. But this requires lot of effort and new knowlege to be gained.   

Workaround: to generate natives as post compile and before Run step.  
`RoboVmBeforeRunTaskProvider` was added that handles this case. If compilation step was ignored -- native code generation will be handled there. It also appears in run configuration as bellow:
![]({{ "/assets/2019/8/15/idea-before-run-step.png" | absolute_url}})  

New Run configuration will automaticaly receive it, existing one has to be added with this step manualy.  

## Not complete/ideal fix
However it is expected to be working most of the time there are few moments to note:  
### Newly created project will fail to build with message:  
```
[ERROR] Couldn't compile app
org.robovm.compiler.CompilerException: Main class com.mycompany.myapp.Main not found
```

This happens as AndroidStudio doesn't provide class path. To fix it -- manualy refresh gradle project or close/open it.

### Previous workaround logic missing   
There was logic in compilation step that additionaly included class directories that were not provided by Idea/AndroidStudio. This logic is not available in case of Run step workaround. But it SHALL not be required. If you have such kind of project to demonstrate the issue please contact me. 

## Only Gradle in case of AndroidStudio
Android studio doesn't ship Maven plugin anymore and Maven managed projects are not possible anymore. Also projects that are managed purely by AndroidStudio are bad idea as Module configuration is Android oriented and doesn't allow to manipulate/configure modules. As result when running plugin in AndroidStudio there will be no Gradle/Maven/None options anymore and Gradle based project will be created.

## Other changes 
* Fixes cancelation logic: once compilation was canceled it only interrupted native code generation for class files but linker was invoker and later failed with tonns of error messages;
* Cosmetic code clean up and migration to recent collection API/lambdas.





