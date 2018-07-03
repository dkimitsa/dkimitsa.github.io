---
layout: post
title: 'bugfix #309: broken step-in/step-over in eclipse'
tags: [fix, debugger, eclipse]
---
**History** [Cannot Use Step Filters while Debugging in Eclipse #309](https://github.com/MobiVM/robovm/issues/309)  
**Fix** [PR315](https://github.com/MobiVM/robovm/pull/315)  

Eclipse plugin is on low priority and it debugger there was working on simple cases year ago I was checking. Simple debug session discovered the bug in Event request validation.  
## Root case
Problem case there was validation of `EXCEPTION_ONLY.referenceTypeID` event modifier against list of known to debugger ones, but there is case when it can be zero:  
<!-- more -->
> referenceTypeID: exceptionOrNull. Exception to report. Null (0) means report exceptions of all types. A non-null type restricts the reported exception events to exceptions of the given type or any of its subtypes.  

The fix is trivial:

```diff
                 case JdwpConsts.EventModifier.EXCEPTION_ONLY:
                     itemId = ((EventExceptionPredicate) predicate).refTypeId();
-                    if (delegates.state().classRefIdHolder().objectById(itemId) == null)
+                    if (itemId != 0 && delegates.state().classRefIdHolder().objectById(itemId) == null)
                         throw new DebuggerException(JdwpConsts.Error.INVALID_CLASS);
                     break;
```

## Protocol version downgraded v1.6 -> v1.5
RoboVM doesn't support bunch of capabilities of v1.6 but there are functionalities that can not be disabled in response to capabilities requests and has to be supported. And eclipse uses it. For example it actively uses `METHOD_EXIT_WITH_RETURN_VALUE` event kind on step-in/step-over filters.  
JDPW version downgrade shall not affect debugger functionality as most of 1.6 features are not supported.
