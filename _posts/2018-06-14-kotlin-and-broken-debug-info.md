---
layout: post
title: 'Kotlin: Local variable resolution is not easy as LocalVariableTable often broken'
tags: [debugger, kotlin, soot, jvm]
---
Keep working on local variable resolution for Kotlin and while having a progress there a bunch of WTF cases related to debug information that is included in Kotlin. Happily it is related to synthetic variable that are inserted by Kotlin, usually it goes into following cases:
1. variable defined to be in some slot within specific bytecode range but this slot is not populated;
2. same as first but in specified local variable slot there is leftover data and it is has not compatible type.

Here is bytecode that demonstrates issue 2:   
<!-- more -->
>org.jetbrains.kotlin/kotlin-stdlib/1.2.41/7e34f009642702250bccd9e5255866f408962a05/kotlin-stdlib-1.2.41.jar  
>class: kotlin.text.Regex:

```
> javap -cp ~/.gradle/caches/modules-2/files-2.1/org.jetbrains.kotlin/kotlin-stdlib/1.2.41/7e34f009642702250bccd9e5255866f408962a05/kotlin-stdlib-1.2.41.jar  -c -l -p -v kotlin.text.Regex

public final java.util.Set<kotlin.text.RegexOption> getOptions();
   Code:
     stack=4, locals=6, args_size=1
        0: aload_0
        1: getfield      #35                 // Field _options:Ljava/util/Set;
        4: dup
        5: ifnull        11
        8: goto          70
       11: pop
       12: aload_0
       13: getfield      #13                 // Field nativePattern:Ljava/util/regex/Pattern;
       16: invokevirtual #39                 // Method java/util/regex/Pattern.flags:()I
       19: istore_1
       20: ldc           #41                 // class kotlin/text/RegexOption
       22: invokestatic  #47                 // Method java/util/EnumSet.allOf:(Ljava/lang/Class;)Ljava/util/EnumSet;
       25: astore_2
       26: aload_2
       27: astore_3
       28: aload_3
       29: checkcast     #49                 // class java/lang/Iterable
       32: new           #51                 // class kotlin/text/Regex$fromInt$$inlined$apply$lambda$1
       35: dup
       36: iload_1
       37: invokespecial #55                 // Method kotlin/text/Regex$fromInt$$inlined$apply$lambda$1."<init>":(I)V
       40: checkcast     #57                 // class kotlin/jvm/functions/Function1
       43: invokestatic  #63                 // Method kotlin/collections/CollectionsKt.retainAll:(Ljava/lang/Iterable;Lkotlin/jvm/functions/Function1;)Z
       46: pop
       47: nop
       48: aload_2
       49: checkcast     #65                 // class java/util/Set
       52: invokestatic  #71                 // Method java/util/Collections.unmodifiableSet:(Ljava/util/Set;)Ljava/util/Set;
       55: dup
       56: ldc           #73                 // String Collections.unmodifiable… == it.value }\n        })
       58: invokestatic  #26                 // Method kotlin/jvm/internal/Intrinsics.checkExpressionValueIsNotNull:(Ljava/lang/Object;Ljava/lang/String;)V
       61: astore_1
       62: aload_1
       63: astore_2
       64: aload_0
       65: aload_2
       66: putfield      #35                 // Field _options:Ljava/util/Set;
       69: aload_1
       70: areturn
     LocalVariableTable:
       Start  Length  Slot  Name   Signature
          28      19     3 $receiver$iv   Ljava/util/EnumSet;
          28      19     4 $i$a$1$apply   I
          20      41     1 value$iv   I
          20      41     5 $i$f$fromInt   I
          64       5     2    it   Ljava/util/Set;
          64       5     3 $i$a$1$also   I
           0      71     0  this   Lkotlin/text/Regex;
```

Problematic case here is following local variable: `64       5     3 $i$a$1$also   I`.  
As it defined its bytecode scope 64:64+5 and it occupies slot 3. Here is a code it reference to:
```
64: aload_0
65: aload_2
66: putfield      #35                 // Field _options:Ljava/util/Set;
69: aload_1
```

And from this code it seen that there is no operation happening with slot 3, so debugger will pick up the value that was stored before. Looking for slot 3 operations and finding following:  
```
22: invokestatic  #47                 // Method java/util/EnumSet.allOf:(Ljava/lang/Class;)Ljava/util/EnumSet;
25: astore_2
26: aload_2
27: astore_3
```

Yep slot 3 will keep reference to `Ljava/util/EnumSet;` object and its type is not compatible with type of `$i$a$1$also` variable.

## Bottom line
While `LocalVariableTable` data built by Kotlin is a mess and makes not simple to resolve local variables for RoboVM case there is a solution: I will completely ignore variables, that contains '$' in name. Anyway, [The $ sign should be used only in mechanically generated source code or, rarely, to access pre-existing names on legacy systems](https://docs.oracle.com/javase/specs/jls/se8/html/jls-3.html#jls-3.8).  

Just in case have create issue at Jetbrains [KT-24932](https://youtrack.jetbrains.net/issue/KT-24932)

## Additional info
Byte code above corresponds to following Kotlin code:
```kotlin
    public val options: Set<RegexOption> get() = _options ?: fromInt<RegexOption>(nativePattern.flags()).also { _options = it }
```
