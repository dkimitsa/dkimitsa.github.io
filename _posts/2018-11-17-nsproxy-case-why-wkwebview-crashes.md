---
layout: post
title: "fix #336: NSProxy case - why WKWebView crashes on didReceiveAuthenticationChallenge"
tags: [bug, binding]
---
Follow up to [issue #336: Could not find Java class corresponding to Objective-C class: WKNSURLAuthenticationChallenge](https://github.com/MobiVM/robovm/issues/336).  
**Short story**  
WKWebView passes instance of internal/unexposed class `WKNSURLAuthenticationChallenge` as parameter instead of expected `NSURLAuthenticationChallenge`, and `WKNSURLAuthenticationChallenge` doesn't extends from expected type.  
Why it works? Because `WKNSURLAuthenticationChallenge` is NSProxy.  

**Long story**  
<!-- more -->
`WKNSURLAuthenticationChallenge` extends from `WKObject` which extends `NSProxy`. All these are not known by cocoa bindings so the crash is due it.  
It works in obj-c side due nature of method invoking -- through sending messages. And NSProxy provides ability to process any selector being sent to it instance. And `WKNSURLAuthenticationChallenge` handles all methods that to be provided by `NSURLAuthenticationChallenge`.

**Solution**  
[pushed as part of io12 bindings](https://github.com/MobiVM/robovm/pull/335/commits/05b08d640efd7eef198c8f3080ed673696e94e21)  
NSProxy concept is not supported by RoboVM and compiler generate code that expects exact type match. This case solution is straight forward workaround:  
- declare `WKNSURLAuthenticationChallenge` class to allow obj-c runtime to resolve it during marshaling;
- inherit it from `NSURLAuthenticationChallenge` to allows it apis to be called over `WKNSURLAuthenticationChallenge` instance.  

```java
@Library("WebKit") @NativeClass
public class WKNSURLAuthenticationChallenge extends NSURLAuthenticationChallenge {
}
```

**Side effects**  
In long term this solution can be broken as `WebKit` can be updated and its logic changed. There might be following problems:
- logic of `didReceiveAuthenticationChallenge` is changed and `WKNSURLAuthenticationChallenge` is not passed there anymore;
- `WKNSURLAuthenticationChallenge` exposed as stand alone public class which makes name clash with workaround.

**Bottom line**  
There is no common solution for NSProxy cases when binding. But approach is to substitute problematic class with surrogate class to fool runtime. It is good that NSProxy is usually internal objects and not exposes outside. 
