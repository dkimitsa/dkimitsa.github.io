---
layout: post
title: 'RoboVM: Workaround for enhance.co'
tags: [bug, hacking, robopods]
---

As continue of [previous post]({{ site.baseurl }}{% post_url 2018-11-05-why-enhance-co-fails-on-robovm %}) here is a workaround:  
Solution is to keep static linking to `Enhance` class from `libconnector.a`. To achieve result lets introduce objc class that extends(and as result statically links) from `Enhance`:   
<!-- more -->
[(all source codes at github)](https://github.com/dkimitsa/codesnippets/tree/master/enhancew)   
Simple XCode static library project was cleated with single wrapper class:  
```objc
#import <Foundation/Foundation.h>
#import "enhance.h"

@interface EnhanceW : Enhance
@end
```

Compile and pack everything with [build.sh](https://github.com/dkimitsa/codesnippets/blob/33e233facda51538a43b695b867bd094faba089c/enhancew/build.sh) script into fat library `libenhancew.a` and it is ready to be used with RoboVM.  
Bindings needs to be manually updated to point to wrapper `EnhanceW` class:
```java
/*<annotations>*/@Library(Library.INTERNAL)
@Marshaler(NSString.AsStringMarshaler.class)
@NativeClass("EnhanceW")
/*</annotations>*/
/*<visibility>*/public/*</visibility>*/ class /*<name>*/Enhance/*</name>*/
    extends /*<extends>*/NSObject/*</extends>*/
    /*<implements>*//*</implements>*/ {
...
}
```   

NP: Don't forget to update <lib> reference in robovm.xml from `libconnector.a` into  `libenhancew.a`
