---
layout: post
title: 'Framework target: fixing back JNI support'
tags: ['target-framework', 'tutorial']
---
[Previous rework]({{ site.baseurl }}{% post_url 2020-02-21-framework-target-improved %}) broke JNI. Rework added logic to start JVM automatically once framework is loaded to provide ObjectiveC classes from Java side available as soon as possible. A bad thing is that JVM structures required for JNI call were not exposed. A try to initialize(second) JVM with `JNI_CreateJavaVM` will fail.  
Goal of the fix is to provide ability to have both JNI and ObjectiveC framework operational. 

## Disabling automatic JVM startup
This can be controlled with another key to `robovm.xml`. Instead, automatic JVM startup will be disabled if `JNI_CreateJavaVM` is present in the list of exported symbols:
```xml
<exportedSymbols>
    <symbol>JNI_CreateJavaVM</symbol>
</exportedSymbols>
```

Compiler will generate(only in case of Framework target) `_bcFrameworkSkipJavaVMStartup` symbol that will be recognized by native code and JVM start up will be skipped. 

## Interoperability 
Once JVM is created it is a good idea(but subject for JNI code design) to ping back `framework support` code to pre-load all objective-c custom classes. In this case JNI based code should call following function:
```c
void rvmInitializeFrameworkWithJVM(JavaVM* externalVm, JNIEnv *externalEnv);
```

## Code 
Code was delivered as [PR497](https://github.com/MobiVM/robovm/pull/497)

