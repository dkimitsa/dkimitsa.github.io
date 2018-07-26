---
layout: post
title: 'Fixing extremely slow IntelliJ Idea UI after migrating MacMini -> MacPro(mid 2012)'
tags: [idea]
---
Just have migrated from Mac Mini to MacPro (4 core xeon model mid 2012) to double number of horses. Just swapped SSD from MacMini into MacPro and it is up and running. Build is almost twice faster now, everything seems to be ok but IntelliJ Idea started working extremely slow. Reinstall OS from scratch but it is pure 1 day wasting. It seemed to me that there is no acceleration enabled and reason is not known for me. Probably switch from Intel GPU to ATI might affect existing system setup. Did find that `apple.awt.graphics.UseQuartz=true` system properties shall enable Quartz acceleration in Java as it is off by default (?). Quick test in Idea and it solves everything !   
To add it:
* open Help->Edit Custom VM Options
* add `-Dapple.awt.graphics.UseQuartz=true` line there  
my working custom options looks like bellow now:

```
-Xms128m
-Xmx1750m
-XX:ErrorFile=$USER_HOME/java_error_in_idea_%p.log
-XX:HeapDumpPath=$USER_HOME/java_error_in_idea.hprof
-Dapple.awt.graphics.UseQuartz=true
```
