---
layout: post
title: 'RoboVM gradle plugin maintenance'
tags: [fix, gradle]
---
RoboVM gradle plugin maintenance [PR289](https://github.com/MobiVM/robovm/pull/289), in it:  

1. added dependency to `build` task to force java compilation for every [RoboVM tasks](https://github.com/robovm/robovm-gradle-plugin). Otherwise commands like `launchIOSDevice` will fail as java classes are not generated.
2. added description to tasks:  
```
> gradle tasks
MobiVM tasks
------------
createIPA - Creates .ipa file. This is an alias for the robovmArchive task
launchConsole - Runs a console app
launchIOSDevice - Runs your iOS app on a connected iOS device.
launchIPadSimulator - Runs your iOS app in the iPad simulator
launchIPhoneSimulator - Runs your iOS app in the iPhone simulator
robovmArchive - Compiles a binary, archives it in a format suitable for distribution and saves it to build/robovm/
robovmInstall - Compiles a binary and installs it to build/robovm/
```  
3. added logic not extracting same file on every launch and do clear cache when new files extracted just to avoid bug cases when incompatible data stays in cache, [check post by @dthommes](https://gitter.im/MobiVM/robovm?at=5ae44a7415c9b0311445a395). Similar as it was done for [Idea plugin]({{ site.baseurl }}{% post_url 2018-04-30-fix286-fix-285-clear-cache-on-new-snapshot %}).  
4. added support for launching debugger, similar as @dukescript did for ['maven plugin'](https://github.com/MobiVM/robovm/pull/258):  
```
.. start with
> gradle --no-daemon -i -Probovm.debug=true -Probovm.debugPort=7778 -Probovm.arch=x86_64 "-Probovm.device.name=iPhone SE" launchIPhoneSimulator
.. attach with
> jdb -attach 7778
```  
