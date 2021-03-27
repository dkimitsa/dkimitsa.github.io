---
layout: post
title: 'fix #567: GlobalValueEnumeration crash'
tags: ['fix']
---
## Background
[issue #567](https://github.com/MobiVM/robovm/issues/567).  
[not exact issue but same problem](https://github.com/MobiVM/robovm/issues/548).  
In general native-to-Java enum mapping is fragile in RoboVM as most likely app will crash if set of enum items in app differs from one available in system:  
<!-- more -->
## case 1: global value is not available on target os   
(case as described in MobiVM#567) -- fails with `UnsatisfiedLinkError`. Happens when app was built for newer API version but run on old device. As result global value is not available in the binary.  
### Root case: native to java marshaller code looks like bellow `valueOf`:
```
for (GlobalValueEnumeration v : values) {
  if (v.value().equals(value)) {
    return v;
}
```
If entry is missing -- v.value() will cause `UnsatisfiedLinkError`

### the fix:
code to be added to check if `v.isAvailable()` before doing `v.value().equals(value)`

## case 2: value is unknown for application. 
Happens when marshalling value returned by system API and it is not known for application. Happens when old app runs on new OS. In this case same `valueOf` marshaller will not find the value in known `values` list and throw `IllegalArgumentException("No constant with value")`.  

### the fix:
It is against enum concept (e.g. it has to be fixed runtime but `GlobalValueEnumeration` is not a true enum) - the fix is to dynamically create new entry of enumeration item and add it to `values`. 

## code 
Code was proposed as [PR571](https://github.com/MobiVM/robovm/pull/571)

## discussion 
This PR updates only `AVMetadataObjectType` but in general all bindings for `GlobalValueEnumeration` to be updated
