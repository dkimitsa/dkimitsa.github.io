---
layout: post
title: 'bugfix #171 - Debugger in eclipse, source code lookup'
tags: [eclipse, debugger, fix]
---
**History** [Add debugging support to Eclipse #171](https://github.com/MobiVM/robovm/issues/171)  
**Fix** [fixed source lookup problem for eclipse. #223](https://github.com/MobiVM/robovm/pull/223)  

Problem in JdwpRefTypeSourceFileHandler.Handle():52 as it was generating full file path. But [by spec](https://docs.oracle.com/javase/7/docs/platform/jpda/jdwp/jdwp-protocol.html#JDWP_ReferenceType_SourceFile) it is shell not.
<!-- more -->
What eclipse does: [String org.eclipse.jdi.internal.ReferenceTypeImpl.getPath(String sourceName)](https://github.com/eclipse/eclipse.jdt.debug/blob/600c6b52bb01fcfa7cabf16261937642dfff5a74/org.eclipse.jdt.debug/jdi/org/eclipse/jdi/internal/ReferenceTypeImpl.java#L1709)

- extracts package from class name
- adds it as prefix to sourceName received from Debugger

As sourceName (from debugger) already contained full package path prefix this causes double package name prefix in the file path. And eclipse was not able to find this path even if proper source folder was manually provided to it
