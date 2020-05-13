---
layout: post
title: 'fixed: idea debug session hang + launcher classes rework'
tags: [fix, idea]
---
# Issue  
There was a rare bug when debug session in Intellij Idea failed and hang. It was not possible to terminate it without reopening the project. 

# Root case  
Issue was reproduced while by having broken deployment to simulator step that allowed to investigate and locate the root case.  
The root case -- Idea hangs on waiting for `stdout`/`stderr` to be closed on terminated (due fail to deploy to sim) process.  
Root case 1: the way stdout was redirected:  
> stdout -> FIFO FILE -> input stream  

Root case 2: wrapper process class `ProcessProxy` around launcher process that was never returned stream to Idea but was never closing it (here it hangs).

# The fix: launcher classes rework 
- Currently, there are three launchers: ios simulator, ios device, console. All use own launcher. The fix introduced common functionality as separated `AbstractLauncherProcess` class. Specific implementations just extends it;
- FIFO and redirect logic massively simplified. No FIFO, just simple Piped streams. Added observable stream to allow a plugin to monitor `stdout` (as part of FIFO replacement) - required by JUNIT plugin;

# Console target fixes
- fixed not working debugging of console target;
- added support for STDIN pipe to console target;
- console template fixed to include required `<target>console</target>`.

# Code 
[PR481](https://github.com/MobiVM/robovm/pull/481) 
