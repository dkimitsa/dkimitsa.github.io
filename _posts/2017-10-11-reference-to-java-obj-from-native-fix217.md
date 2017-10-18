---
layout: post
title: 'bugfix #217 - Reference to java object is not maintained by objc'
tags: [gc, fix]
---
**History** [AReference to java object is not maintained by objc, when the java object is garbage collected #217](https://github.com/MobiVM/robovm/issues/217)  
**Fix** [PR fixed #217 #219](https://github.com/MobiVM/robovm/pull/219)  

The problem is that ObjCObject keep reference of native object pointer (peer) to java object in map where java object is maintained in week reference vo.
As result all java objects that is not referenced at Java side are subject for GC. And this is wrong.
<!-- more -->
This problem was introduced by my PR [Issue 894 fix + partial ios10 binding #101](https://github.com/MobiVM/robovm/pull/101). It was old issue from original RoboVM [#894](https://github.com/robovm/robovm/issues/894).
The issue is relevant for custom classes objects (user native objects subclasses) as before #101 references to Java objects were retained in ObjCObject.ObjectOwnershipHelper in every intercepted native retain call and released in intercepted release call.

The flow was changed due fun moment that was happening in UIKit -- retain often is called after dealloc, here is small example (lines 15, 11)
```
9   ....
10  Main                     [J]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J + 168
11  Main                     [j]org.robovm.objc.ObjCObject$ObjectOwnershipHelper.retain(JJ)J[callback] + 122
12  UIKit                    -[UIView(Hierarchy) _postMovedFromSuperview:] + 513
13  UIKit                    __UIViewWasRemovedFromSuperview + 169
14  UIKit                    -[UIView(Hierarchy) removeFromSuperview] + 512
15  UIKit                    -[UIView dealloc] + 555
16  libdispatch.dylib        _dispatch_client_callout + 8
17  libdispatch.dylib        _dispatch_main_queue_callback_4CF + 1260
18  ....
35  Main                     0x0000000101667b1b main + 27
36  libdyld.dylib            0x000000010c71bd81 start + 1
37  ???                      0x0000000000000005 0x0 + 5
```

Fix returns logic to manage java references in retain/release cycles and adds additional callback handler to dealloc to solve 'retain after dealloc issue'. It just calls supper dealloc(which will produce unexpected retains) and wipes reference after it.
