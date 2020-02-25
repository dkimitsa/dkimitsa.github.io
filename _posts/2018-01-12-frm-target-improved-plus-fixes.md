---
layout: post
title: 'improved Framework target + bunch of fixes'
tags: ['target-framework', fix]
---
Main effort was put provide better support for Framework target. During coding bunch of bugs have been fixed ands several improvements were done. All these changes delivered as [PR253](https://github.com/MobiVM/robovm/pull/253). Complete list of changes:
<!-- more -->

* Fixed: there was no target type checking and it was possible to select Framework module in iOS run configuration;
* [Added](https://github.com/dkimitsa/robovm/commit/6c8358926e5555b9013d1c08dbe7981dc328684b) template for Framework target;
* [Added](https://github.com/dkimitsa/robovm/commit/17db26b35f427de711f77ba4995290524da39122) framework support native library, so there is no need to write native support for Framework anymore (there will be a tutorial link later);
* Framework target is [added](https://github.com/dkimitsa/robovm/commit/452177c62b8ebfde2066f3c8cb514b3fbc4309ef) to RoboVM projects list in Idea plugin;

![]({{ "/assets/2018/01/12/robovm-idea-new-framework-project.png"}})
* Idea plugin: RoboVM related items has been moved to RoboVM menu, also added to menu:
  - clean cache functionality;
  - menu item to bring up "Create framework" dialog;
  - create new Project/Module as workaround for Android Studio;
  ![]({{ "/assets/2018/01/12/robovm-idea-new-menu.png"}})
* [Added](https://github.com/dkimitsa/robovm/commit/257b7fba493969164a54dadd2a1953b572856f19) dialog to build framework target in Idea plugin;
![]({{ "/assets/2018/01/12/robovm-idea-create-framework-dialog.png"}})
* [Fixed](https://github.com/dkimitsa/robovm/commit/0e985e0e92250b083f4591dbe24f1edee24a4759) completely broken “Add new module” functionality in Idea:
  - new module files were unpacked in project directory instead of module one;
  - files are now extracted to folder specified by `content root` parameter;
  - sometimes gradle import was starting before files are extracted to dist (fixed);  
* [Fixed](https://github.com/dkimitsa/robovm/commit/1b5090a7f6a07c38b2f49f41b1ecff47616303d1) bug when launcher was putting `get-task-allow: true` for cases when application was signed with `adhoc` or `enterprise` provisioning profile which caused application signature become invalid;
