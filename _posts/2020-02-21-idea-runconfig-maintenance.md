---
layout: post
title: "Idea plugin: run configuration maintenance"
tags: [fix, idea]
---
## Why
Idea run configuration might become invalid if configured items become invalid, like:
- simulator is not available in next xcode;
- signing identity or provision profile expired or removed;

### Configuration editor was hiding errors
In this case compilation failed with runtime exception. But once run configuration editor is opened instead of pointing to invalid items editor was auto picking up first one available. As result user was not able to see a problem and to have configuration overridden changes to it to be done. Otherwise `apply` button was not active.

### Name instead unique id was used as identifier
To identify signing identity/simulator/provisioning profile its name was saved. For `auto` value just `auto` text placeholder was saved and checked against. This is subject for name collisions.

## Whats changed
### Auto mode explicitly specified
Field that might be auto (Signing identity/Provisioning profile/Simulator) are now have separate parameter in config file that specifies entry type: auto or explicit ID.
And auto mode is identified by it and not string constant in entry name.

### Explicit identifier instead of name
When explicit entry is used (e.g. not auto) entries id is saved instead of entry name (udid for profile/simulator, footprint for signing identity). No collision anymore.

### Auto mode for simulator
![]({{ "/assets/2020/2/21/idea-run-auto-sims.png"}})
Simulator field receives two modes `auto iPhone` and `auto iPad`. These modes acts similarly to gradle's one 'launchIPhoneSimulator'/`launchIPadSimulator`.
Also in case there is no simulator available in system it will allows to recover broken run configuration.

### Quick fix for broken fields
![]({{ "/assets/2020/2/21/idea-run-quick-fix.png"}})
Problem fields can be quick fixed to auto value with `Fix` button.

### arm64 is default for device target
Default `thumbv7` was a problem as is not available since ios11 (on most devices today). This caused deployment error at very end of run cycle. Having `arm64` default improves user experience.

### no more x86 (32bit) option for ios11+ simulators
Similar issue to arm64. It 32bit code doesn't run on ios11+ simulators.

### no more `don't sign` options
`don't sign` option was for usage with jailbroken ios devices. While having free provisioning from Apple this option have no more sense. Also `ldid` binary was removed. If `don't sign` option is specified from gradle/maven ADHOC signing will be performed.

### bonus1: gradle import of template projects was fixed
Created from template gradle projects were broken due:
- gradle was set to use RoboVM SDK (and it is not operational any more for tools);
- Java source code level was set to Java12+;

Now: gradle uses internal JDK, java source level forced to Java8.

### bonus2: added support for `Apple Development` certificates
These were not part of regex and were ignored when `auto` identity was specified.

### bonus3: suppressed -559038737 error code
If simulator process is terminated by Idea strange error message was printed into console log:
>Process finished with exit code -559038737

This code is UNKNOWN_CODE constant in Executor. Now it is being recognized and replaced with 0.

## Code
Code was delivered as [PR456](https://github.com/MobiVM/robovm/pull/456)


