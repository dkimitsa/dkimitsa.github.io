---
layout: post
title: 'bugfix: breakpoints were not working in Kotlin code'
tags: [fix, debugger, kotlin, idea]
---
Previous [debugger maintenance]({{ site.baseurl }}{% post_url 2018-05-23-fix295-crashes-kotlin-debugger %}) allowed to build kotlin code for debug and attach debugger for it. It was even possible to step-in into kotlin code. Absence of any kotlin experience didn't allow to test it deeper. As it kept failing on simplest case, setting of breakpoint, with following message:  
> No executable code found at line XX in class YYYY

Root case of this -- debug plugin and debugger considered JVM code as compiled from Java file and returned `.java` file extension everywhere.  
The fix is bellow:  
<!-- more -->
## Root case
JDWP `ReferenceType(2).SourceFile(7)` request hander in debugger had no information about source file for class and was building it from class name as bellow:
> filename = class_name + .java  

This caused `Idea` to consider things wrong and show `No executable code found at ...` The fix is to return valid file name but at this point debugger had no information for source file name.

## Extending debug information
Solution is to extend existing debug information payload with class source name. This information is available from `soot` during compilation time. Once valid file name returned by `ReferenceType(2).SourceFile(7)` issue is gone.  
Check details in the [commit](https://github.com/MobiVM/robovm/pull/297/commits/b87fbabd80a67c790f7a0d7bf44c8486730351b5)
