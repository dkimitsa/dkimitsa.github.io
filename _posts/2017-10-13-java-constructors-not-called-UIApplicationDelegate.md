---
layout: post
title: 'bugfix #221 - java constructor is not called in UIApplicationDelegate implementation'
tags: [fix, native2java]
---
**History** [java constructor is not called in UIApplicationDelegate implementation #221](https://github.com/MobiVM/robovm/issues/221)  
**Fix** [PR #222](https://github.com/MobiVM/robovm/pull/222)  

UIApplicationDelegate is being created from native side. And it is constructor is not called so all instance members are not initialized. Yep same [#894](https://github.com/robovm/robovm/issues/894). But why?
<!-- more -->
Two reasons:
- constructor callbacks are generated for classes marked with @CustomClass annotation (there is 0% of UIApplicationDelegate has this in code)
- even if it is annotated there is no @Method annotation at default constructor of NSObject so it will be skipped

Fix does:
- adds @Method annotation to NSObject default constructor. But this will require developers to use @CustomClass which is lame and against today tradition not to.
- adds sugar during at compilation level. Compiler will consider any class that implements UIApplicationDelegate as @CustomClass (user has not to) and generate all required callbacks
