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
![]({{ "/assets/2019/8/15/idea-gradle-options.png"}})

## The fix
<!-- more -->
Proper one: assume that is best way is to integrate into gradle build process similar as Fabric does. But this requires lot of effort and new knowlege to be gained.   

Workaround: to generate natives as post compile and before Run step.  
`RoboVmBeforeRunTaskProvider` was added that handles this case. If compilation step was ignored -- native code generation will be handled there. It also appears in run configuration as bellow:
![]({{ "/assets/2019/8/15/idea-before-run-step.png"}})  

New Run configuration will automaticaly receive it, existing one has to be added with this step manualy.  

## Only Gradle in case of AndroidStudio
Android studio doesn't ship Maven plugin anymore and Maven managed projects are not possible anymore. Also projects that are managed purely by AndroidStudio are bad idea as Module configuration is Android oriented and doesn't allow to manipulate/configure modules. As result when running plugin in AndroidStudio there will be no Gradle/Maven/None options anymore and Gradle based project will be created.

## Other changes 
* Fixes cancelation logic: once compilation was canceled it only interrupted native code generation for class files but linker was invoker and later failed with tonns of error messages;
* Cosmetic code clean up and migration to recent collection API/lambdas.





