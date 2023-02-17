---
layout: post
title: 'Seamless XCFramework support'
tags: [technote, compiler]
---

XCFramework is a distributable binary package that contains variants of binary framework or library built for different platforms/cpu architectures. Introduced in Xcode11.
Also, it solves problem packaging Arm64 and Arm64-simulator binaries into Fat framework/library. 
RoboVM had [workaround](https://github.com/MobiVM/robovm/pull/483) for two year already to framework path for different architectures. But it requires manual preparation.copying for each framework and a bit annoying.
[PR694](https://github.com/MobiVM/robovm/pull/694) introduced two options to improve the usability:
## 1. Direct reference in `<xcFrameworks>`
<!-- more -->
New option `<xcFrameworks>` in `robovm.xml` allows to directly list paths (absolute or relative) to xcframworks similar as static libraries are configured:
```
<xcFrameworks>
   <path>libs/first.xcframework</path>
   <path>libs/second.xcframework</path>
</xcFrameworks>
```

## 2. Lookup by name in framework path
XCFrameworks to be mentioned as regular frameworks in `<frameworks>` section of `robovm.xml`.  
Introduced change performs scan of paths specified in `<frameworkPaths>` and looks for xcframework with given name.
Once found -- proceed it as xcframework.
Its a transparent mode compatibility mode that requires no changes to existing configurations and alls keeps compatibility with existing robopods as they have configured dependency in `<frameworks>` section.

This lookup can be disabled in case of bug by new introduced configuration key: To disable `xcframework` logic following might be used:
```
<experimental>
     <xcFrameworkLookup>false</xcFrameworkLookup>
</experimental>
```

## 3. How it works 
When xcframework is found by name specified in `<frameworks>` there are few moments to note:
- library name inside xcframework could be different that name of xcframework;
- xcframwork could contain static libraries instead of frameworks.

As result if framework was resolved to be xcframwork:
- its name is removed from `<frameworks>`;
- if xcframework contains a framework -- it's internal name added to `<frameworks>`, path to matching platform/arch is added `<frameworkPaths>`;
- if its a static library -- its added to `<libs>` list;

All these changes works on `Config.java` class level. Introduced `ResolvedLocations.java` class that performs lookup and prepares `frameworks`, `frameworkPaths` and `libs` list when corresponding items are accessed.  

# Code 
Change was proposed as [PR694](https://github.com/MobiVM/robovm/pull/694)
