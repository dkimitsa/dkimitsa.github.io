---
layout: post
title: 'Intellij Idea plugin: run function + DSL UI preview panel'
tags: ['idea-plugin']
---
While Intellij Idea's code base is migrating to kotlin, its UI is migrated from Swing/AWT to their own [Kotlin UI DSL](https://plugins.jetbrains.com/docs/intellij/kotlin-ui-dsl-version-2.html) implementation similar to Kotlin compose.    
Sadly there is no preview panel available for this kind of code (There was [semi-official](https://plugins.jetbrains.com/plugin/14228-intellij-kotlin-ui-dsl-preview) plugin but it's not supported anymore.    

Let get few hacks to have it available:  
![]({{ "/assets/2025/06/03/screen_ide.png"}})
<!-- more -->

## Idea 
- get compose-like preview function that returns AWT component;
- run it in scope of plugin;
- attach returned result to preview panel.

## Unsafe WARNING  
This approach is extremely unsafe as run code in scope of Intellij Idea/Plugin context.  

## How it works/how to use 
Plugin install LineMarkerProvider that will attach double-run icon (check screenshot above) to any kotlin function that has no parameters.  
Hitting this icon will trigger run cycle that will involve:
- Module, that contains function will be built (same as from Build/Build module menu);
- Class path of module will be collected and ClassLoader will be created for this class pass;
- Class is loaded using reflection;
- Target function is invoked using reflection;
- During the run `System.out`/`System.err` is intercepted and redirected to Console panel;
- If function returns UI element (AWT Component) -- its being attached to preview panel. 


Hope this plugin will be useful as for UI DSL development as for quick-and-dirty function run to check things. 

## Source code and pre-build binary 

Source code and pre-built binaries pushed to stand-alone branch of [code-snippets](https://github.com/dkimitsa/codesnippets/tree/idea/plugin/unsafe-run) repo.

