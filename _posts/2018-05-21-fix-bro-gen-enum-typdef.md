---
layout: post
title: 'bro-gen: fixing enum typedef'
tags: [fix, 'bro-gen']
---
Today bug-fix/maintenance delivers following problem, following such obj-c code:
```objc
typedef enum LogLevelValues
{
    IS_LOG_NONE = -1,
    IS_LOG_INTERNAL = 0,
    IS_LOG_INFO = 1,
    IS_LOG_WARNING = 2,
    IS_LOG_ERROR = 3,
    IS_LOG_CRITICAL = 4,

} LogLevel;

- (void)sendLog:(NSString * )log level:(LogLevel)level tag:(LogTag)tag;
```

is being converted into following broken Java one:  
<!-- more -->
```java
public enum /*<name>*/LogLevelValues/*</name>*/ implements ValuedEnum {
    /*<values>*/
    NONE(-1L),
    INTERNAL(0L),
    INFO(1L),
    WARNING(2L),
    ERROR(3L),
    CRITICAL(4L);
    /*</values>*/
}

public SomeClass {
    @Method(selector = "sendLog:level:tag:")
    public native void sendLog$level$tag$(String log, LogLevel level, int tag);
}
```

So the issue: enum name is generated from enum tag but method uses typedef name, this causes compilation error.

## Fix: is to use only enum tag everywhere
This produce valid java file for such case. All changes in [commit](https://github.com/dkimitsa/robovm-bro-gen/commit/42c896a22ac38384131043e0c0b8c1d83df62f1e).
