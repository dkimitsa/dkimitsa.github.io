---
layout: post
title: 'Crashlytics: fight for exception handler'
tags: [hacking, fix]
---
This post continues tutorial for [proper initialization of crash reporters t]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %}). Today it focus around [issue 350:Crashlytics: Caught exceptions are being reported as crashes](https://github.com/MobiVM/robovm/issues/350).   

## Root case 
`Signals.installSignals()` preserves signals handlers and allows RoboVM to handle NPE. But Crashlytics also uses [mach exception handler](https://www.mikeash.com/pyblog/friday-qa-2013-01-11-mach-exception-handlers.html) to get crash Apple way. These have priority over signals and this allows Crashlytics to see null pointer exception (EXC_BAD_ACCESS) before RoboVM detects it and report. 

## The fix  
<!-- more -->
The fix approach is same as with signals: 
- save exceptions handlers before initializing reporters;
- initialize third party sdks that might install signals/exception handlers;
- restore exceptions handlers.

`Signals.installSignals()` was extended with additional parameter `preservePorts`. When it set to true mach exceptions handlers will be saved and restored; Old API without this parameter is also available and will not save exception handlers by default. Usage of this API following: 
```java
Signals.installSignals(() -> {
    // initialize crash reporters here
    Fabric.with(Crashlytics.class);
}, true);
```

## Code
All changes are delivered as [PR352](https://github.com/MobiVM/robovm/pull/352).  

## Possible problem
This approach doesn't provide chaining of handlers, e.g. if it is crash in native code third party exception handler will not be called.  
In case Library depends on this functionality this workaround can't be used. 