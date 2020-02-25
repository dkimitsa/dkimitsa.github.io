---
layout: post
title: 'bugfix: Debugger hangs when evaluating native objects that has no corresponding Java class loaded'
tags: [debugger, fix]
---
**Fix** [PR #256](https://github.com/MobiVM/robovm/pull/256)  

This happens when evaluating internal of `native` object in debugger (NSDictionary in my case) and bound Java class for internal objects (NSDictionary$EntrySet in my case) was not loaded by Runtime. E.g. class resolution is happening by Debugger itself.
The bug's symptoms are following: app hangs, and variable is not resolved in debugger:
![]({{ "/assets/2018/01/19/debugger-evaluate-exp-bug.png"}})

In idea log it appears as exception:
<!-- more -->

```
13:03:40.257 SEVERE: Debugger: Thread JDWP server socket thread crashed
org.robovm.debugger.DebuggerException: Unable to resolve runtime class for org/robovm/apple/foundation/NSDictionary$EntrySet
	at org.robovm.debugger.state.classdata.RuntimeClassInfoLoader.resolveRuntimeDataTypeInfo(RuntimeClassInfoLoader.java:98)
	at org.robovm.debugger.state.classdata.RuntimeClassInfoLoader.resolveRuntimeDataTypeInfo(RuntimeClassInfoLoader.java:46)
	at org.robovm.debugger.state.classdata.RuntimeClassInfoLoader.resolveObjectRuntimeDataTypeInfo(RuntimeClassInfoLoader.java:60)
	at org.robovm.debugger.delegates.InstanceUtils.instanceByPointer(InstanceUtils.java:411)
	at org.robovm.debugger.delegates.InstanceUtils.instanceByPointer(InstanceUtils.java:438)
	at org.robovm.debugger.delegates.InstanceUtils.jdwpInvokeMethod(InstanceUtils.java:619)
	at org.robovm.debugger.delegates.AllDelegates.jdwpInvokeMethod(AllDelegates.java:349)
	at org.robovm.debugger.jdwp.handlers.objectreference.JdwpObjRefInvokeMethodHandler.handle(JdwpObjRefInvokeMethodHandler.java:52)
	at org.robovm.debugger.jdwp.JdwpDebugServer.doSocketWork(JdwpDebugServer.java:196)
	at org.robovm.debugger.jdwp.JdwpDebugServer.lambda$new$0(JdwpDebugServer.java:129)
	at java.lang.Thread.run(Thread.java:748)
	at org.robovm.debugger.utils.DebuggerThread.run(DebuggerThread.java:42)
```

Root case: runtime signature of object doesn't contain "L" prefix and ";" suffix as it should up to JVM. Lookup in debugger state fails as no ClassInfo for runtime class is found. Solution -- add missing suffix and prefix:
```patch
--- plugins/debugger/src/main/java/org/robovm/debugger/state/classdata/RuntimeClassInfoLoader.java	(revision 39e0c4cf2a65f63f0fe64efe005962204c19a629)
+++ plugins/debugger/src/main/java/org/robovm/debugger/state/classdata/RuntimeClassInfoLoader.java	(date 1516360006000)
@@ -66,9 +66,7 @@
      * @return data type info for giver clazz pointer
      */
     public ClassInfo resolveRuntimeDataTypeInfo(String signature, long clazzPtr) {
-        ClassInfo info = delegates.state().classInfoLoader().classInfoBySignature(signature);
-        if (info != null)
-            return info;
+        ClassInfo info;

         // try to build array data type
         char firstChar = signature.charAt(0);
@@ -87,6 +85,11 @@

             info = new ClassInfoArrayImpl(signature, componentType);
         } else {
+            // it could be class, signature from device it is Clazz and not bounded by L and ;
+            info = delegates.state().classInfoLoader().classInfoBySignature("L" + signature + ";");
+            if (info != null)
+                return info;
+
             // check for dynamically created proxy classes
             if (Pattern.matches(".*/\\$Proxy\\d+$", signature)) {
                 // it is proxy, return simple java object
```

Bottom line: how this bug was living so long ?
