---
layout: post
title: 'bugfix #263: Idea launches wrong version of simulator than selected in run configuration'
tags: [fix, idea]
---
**History** [robovm + idea: wrong simulator version is launched than selected in run configuration #262](https://github.com/MobiVM/robovm/issues/262)  
**Fix** [PR263](https://github.com/MobiVM/robovm/pull/263)  

It is kind of annoying bug usually not seen as recent version of simulated hardware and SDK is used. But if you have to specify exact version of SDK it is pain. Fix is trivial -- include version check in saved Simulator configuration:  
<!-- more -->
(Just one of several places changed)  
```diff
-        // set simulator types
+        // set simulator types and find one that matches version and name
+        int exactSimVersonMatchIdx = -1;
+        int bestSimNameMatchIdx = -1, bestSimNameMatchVersion = -1;
+        int bestDefaultSimMatchIdx = -1, bestDefaultSimMatchVersion = -1;
         for (DeviceType type : DeviceType.listDeviceTypes()) {
             simType.addItem(new SimTypeWrapper(type));
-            if (type.getDeviceName().equals(config.getSimulatorName())) {
-                simType.setSelectedIndex(simType.getItemCount() - 1);
-            } else if (config.getSimulatorName().isEmpty() && type.getDeviceName().contains("iPhone-6") && !type.getDeviceName().contains("Plus")) {
-                simType.setSelectedIndex(simType.getItemCount() - 1);
+            if (type.getDeviceName().equals(config.getSimulatorName()) && type.getSdk().getVersionCode() == config.getSimulatorSdk()) {
+                exactSimVersonMatchIdx = simType.getItemCount() - 1;
+            } else if (type.getDeviceName().equals(config.getSimulatorName()) && type.getSdk().getVersionCode() > bestSimNameMatchVersion) {
+                bestSimNameMatchIdx = simType.getItemCount() - 1;
+                bestSimNameMatchVersion = type.getSdk().getVersionCode();
+            } else if (config.getSimulatorName().isEmpty() && type.getDeviceName().contains("iPhone-6") &&
+                    !type.getDeviceName().contains("Plus") && type.getSdk().getVersionCode() > bestDefaultSimMatchVersion) {
+                bestDefaultSimMatchIdx = simType.getItemCount() - 1;
+                bestDefaultSimMatchVersion = type.getSdk().getVersionCode();
             }
         }
+        if (exactSimVersonMatchIdx < 0) {
+            // if exact match is not found use name match or default simulator
+            if (bestSimNameMatchIdx >= 0)
+                exactSimVersonMatchIdx = bestSimNameMatchIdx;
+            else if (bestDefaultSimMatchIdx >= 0)
+                exactSimVersonMatchIdx = bestDefaultSimMatchIdx;
+            else exactSimVersonMatchIdx = simType.getItemCount() - 1;
+        }
+        if (exactSimVersonMatchIdx >= 0)
+            simType.setSelectedIndex(exactSimVersonMatchIdx);
```
