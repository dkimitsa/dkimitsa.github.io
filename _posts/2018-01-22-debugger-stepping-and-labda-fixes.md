---
layout: post
title: 'bugfix: Debugger - broken step over and crashed when using labda'
tags: [debugger, fix]
---
**Fix** [PR #257](https://github.com/MobiVM/robovm/pull/257)  

Wants find bugs in own product ? Start using it!  
## 1. It was not possible to step over code if this code was generating exception (even if it was handling it)
These lines were not possible to step over as in `exists()` there was exception generated/catch:
```java
new File("/some/invalid/path").exists();
new File("/some/invalid/path").exists();
```
<!-- more -->

*Root case*: once VM reported exception it also removed stepping constraints from target, and JDWP implementation was not restoring when handling these event.  
*Solution*: restore constraints if exception is ignored and not reported to IDE(jdwp client);


## 2. Debugger was incorrectly resolving scope of local variables if their usage was starting with invoking of lambda
This bug caused application to crash when debugging the following example:
```java
private void startParsing() {
	List<File> cachesToParse = new ArrayList<>();
	File arm32cache = new File("/System/Library/Caches/com.apple.dyld/dyld_shared_cache_armv7s");
	File arm64cache = new File("/System/Library/Caches/com.apple.dyld/dyld_shared_cache_arm64");

    // breakpoint at this line crashed the app
	if (arm32cache.exists())
		cachesToParse.add(arm32cache);
	if (arm64cache.exists())
		cachesToParse.add(arm64cache);
	for (File cacheFile : cachesToParse) {
		try {
			DyLdCache cache = new DyLdCache(cacheFile);
			cache.readImages((image, imageIdx, imageCnt) -> NSOperationQueue.getMainQueue().addOperation(() -> {
				label.setText("Reading: (" + imageIdx + "/" + imageCnt + ") " + image );
			}));
		} catch (MachOException e) {
			e.printStackTrace();
		}
	}
}
```

*Root case*: variable `cache` was incorrectly resolved to have scope starting at first line of the method. And at moment of breakpoint debugger was trying to read value of `cache` variable but it was not defined yet. That caused BAD_ACCESS exception and crash. This was happening due following scenario:
- RoboVM implements Lambdas as external classe and replaces call to it with invocation of static method of newly generated class;
- the variable `cache` has first unit usage set to lambda invocation but this code was replaced with generated one;
- generated code doesn't contain line number information and this caused debugger to consider this variable to be scoped from first line of method;  
*Solution*: add line number information to generated code same as it was in original lambda invocation.
