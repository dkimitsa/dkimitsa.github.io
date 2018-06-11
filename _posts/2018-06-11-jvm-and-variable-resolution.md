---
layout: post
title: 'Debugger: variable resolution for Kotlin'
tags: [debugger, kotlin, soot, jvm]
---
Short update on debugger support for `kotlin`.  
Recent fixes ([#1]({{ site.baseurl }}{% post_url 2018-05-23-fix295-crashes-kotlin-debugger %}), [#2]({{ site.baseurl }}{% post_url 2018-05-25-fixing-debugger-kotlin-case %})) unblocked debugger for kotlin but variable resolution is still broken. It still produces following error messages on synthetic `kotlin` code for lambdas and collections:  
> [ERROR] Unable to resolve variable $i$a$1$forEach in method test, class com.test.ViewController

This happen as this variable was declared in synthetic code and this code is out scope defined in debug information for this variable.  
Code for variable resolution was pending to upgrade long time ago and best case would be to have variable information without dependency to debug information.  
But not so fast: it is not always possible to do it straight away without assumptions and debug information. Integer types case is described bellow.  
<!-- more -->

## Soot
RoboVM uses [Soot](https://github.com/Sable/soot) to decode java `.class` file into `Jimple` pseudo-code, which is later being turned into LLVM IR language. Variable resolution directly depends on how Soot decodes compiled java code and how it resolves variables. Here goes the problem.

## The problem

```java
public static short s;
public static int intTest() {
    int ii = s;
    ii += 1;
    return ii;
}
```

This code is being decompiled from `class` into following `Jimple` :  
```
public static int test()
{
    short s1;
    int i2;
    s1 = <com.test.BugTest: short s>;
 label0:
    i2 = s1 + 1;
    return i2;
//    localvar index=0 name=ii type=I start=label0 end=<end_of_method>;
}
```

What wrong here:
* var `ii` declared to be available at `label0` but there is no value attached to it yet (value is in s1);
* value located in variable s1 (type short) that is not part of source code;
* both variable `ii` and `s1` refers to same local variable slot `index 0` which breaks variable resolution (as only one variable shall in local variable slot at one time);

## JVM
Java produce following bytecode for snippet above:
```
>javap.exe -cp . -bootclasspath . -c -l -p com.test.BugTest 

public static int intTest();
  Code:
     0: getstatic     #2                  // Field s:S
     3: istore_0
     4: iinc          0, 1
     7: iload_0
     8: ireturn
```

In JVM both operand stack and local variable slots are `word size` e.g. 32 bit which is enough to hold all integer types but not long (which takes 2 word). From byte code above short value is being stored into local variable slot 0 with instruction `istore_0` and it is stored as integer, and later it is being manipulated as integer. This split happened as `soot` performs local split on each assignments before type resolution. Then attach type based on assignments and after this performs local packing. But in case variables after split have different type -- it results to variable not being joined back.

## RoboVM approach
RoboVM hacked Soot's LocalPacker code to not split variables based on debug information so this `ii` variable to to be assigned to `int` type.
This approach works well as long as debug information matches byte code and locals a being used in scope of this debug information. But it is goes broken with injected synthetic code such kotlin one.

## How it works with Java
it is not an issue with java as all variables are stored in local variable slots, and are converted to integers. As long as debugger needs a variable it just ask JVM to return variable of type XXX from slot YYY.

## Extra: decompiling with Fernflower
Case 1: without debug information -- same as soot does:
```
public static int intTest() {
    short var0 = s;
    int var1 = var0 + 1;
    return var1;
}
```

Case 2: with debug information:
```
public static int intTest() {
    int ii = s;
    int ii = ii + 1;
    return ii;
}
```

## Conclusion
Work in progress, stay tuned
