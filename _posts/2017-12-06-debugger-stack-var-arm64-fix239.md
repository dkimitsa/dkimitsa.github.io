---
layout: post
title: 'bugfix #239 - Debugger hangs on ARM64 due resolving locals bug'
tags: [debugger, fix]
---
**History** [Debugger hangs due exception on ARM64 target when trying to resolve stack variables #239](https://github.com/MobiVM/robovm/issues/239)  
**Fix** [PR #240](https://github.com/MobiVM/robovm/pull/240)  

Have almost never debugged ARM64 target (always used thumb7 as faster and smaller), but turned out that debugger is pretty broken. The reason is forgotten case:  
<!-- more -->
Register `OP_breg31 (RSP)` on ARM64 was not handled completely, but has to be handled same way as `OP_breg13 (SP)` on thumb7 with respect of spfp offset. So fix is obvious:
```diff
--- plugins/debugger/src/main/java/org/robovm/debugger/delegates/StackFrameDelegate.java	(revision e913c0aa3a6312e6304911a92a8011c2309d2908)
+++ plugins/debugger/src/main/java/org/robovm/debugger/delegates/StackFrameDelegate.java	(revision )
@@ -224,8 +224,9 @@
         if (variableInfo.register() == DebugVariableInfo.OP_fbreg) {
             // FP register on x86 and ARM64
             addr = frame.fp() + variableInfo.offset();
-        } else  if (variableInfo.register() == DebugVariableInfo.OP_breg13) {
-            // SP register on ARM 32bit
+        } else  if (variableInfo.register() == DebugVariableInfo.OP_breg31 || variableInfo.register() == DebugVariableInfo.OP_breg13){
+            // SP on thumb7(R13)
+            // RSP register on ARM64
             // align using SpFpOffset data
             addr = (frame.fp() - frame.methodInfo().spFpOffset()) & ~(frame.methodInfo().spFpAlign() - 1);
             addr += variableInfo.offset();
```

It is strange that nobody had reported this before. 
