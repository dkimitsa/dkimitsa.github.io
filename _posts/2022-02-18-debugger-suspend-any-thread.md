---
layout: post
title: 'Debugger -- suspend any thread'
tags: [debugger, technote]
---
RoboVM code can be stopped only when code reach instrumented hook callbacks (ones that being injected after every line by compiler). As result pausing application will in most cases not stop the application and will not show valid call stack:
![]({{ "/assets/2022/02/18/debugger-wrong-thread-state.png"}})

### Root case
- Thread state is not delivered from VM to debugger so always reported as running;
- "Paused" thread doesn't show any call stack due no ability to pause it other places than instrumented hooks code;
 
### Approach
<!-- more -->
Delivering thread state is trivial as it is available at JVM level:
![]({{ "/assets/2022/02/18/debugger-proper-states.png"}})

Stopping thread is a bit tricky.  
Interrupting thread and running code on top of it is possible by sending a signal to it. RoboVM is already uses two user signals available and only the option is to reuse these. 
One is used for SocketIO operations and second for thread interruption to retrieve it stack trace to implement `Thread.getStackTrace()` API.     

Its perfect match as does exactly two things needed: 
- interrupts thread and allow code to be run on top of it call stack;
- returns call stack.

### Implementation 
interrupt any random thread is possible by sending a signal to it. There is already two signals dedicated to RoboVM and introducing another one would be wastefully.
Lucky for us one signal was used to capture thread stack trace. Last ones also good to have when you suspend the thread. So it was extended to provide a callback to hooks infrastructure and if thread was pending for suspend -- suspend loop was running there. Debugger code was updated to support these changes in debugger-hooks protocol.
![]({{ "/assets/2022/02/18/debugger-sleeping-with-stack-trace.png"}})

Changes proposed as [PR628](https://github.com/MobiVM/robovm/pull/628)
