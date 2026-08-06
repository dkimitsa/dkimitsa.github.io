---
layout: post
title: 'Java 8: investigating a NullPointerException in a default method'
tags: [bug]
---
While working on [compose for RoboVM]({{ site.baseurl }}{% post_url 2026-08-05-compose-and-robovm %}), a strange NPE appeared:
>  java.lang.NullPointerException
>      at androidx.compose.runtime.MonotonicFrameClock.getKey(MonotonicFrameClock.kt)

The simplified code looks like this:
```kotlin
  public interface MonotonicFrameClock : CoroutineContext.Element {
      override val key: CoroutineContext.Key<*>
          get() = Key
      public companion object Key : CoroutineContext.Key<MonotonicFrameClock>
  }
```

It was clear that the issue was related to a default method in an interface, but let's reduce it to a simple Java sample to eliminate Kotlin overhead:
<!-- more -->
```java
public interface MyInterface {
  Object MY_STATIC = new Object() {};
  default Object getMyStatic() { return MY_STATIC; }
}

public class MyImpl implements MyInterface {
}

void test() {
  MyInterface obj = new MyImpl();
  Object val = obj.getMyStatic();
}
```

## Root cause:
As per [Java Virtual Machine Specification, 5.5](https://docs.oracle.com/javase/specs/jvms/se22/html/jvms-5.html#jvms-5.5), static fields in an interface are initialized when the interface itself is initialized, which happens when:
> A class or interface C may be initialized only as a result of:
> - If C is an interface that declares a non-abstract, non-static method, the initialization of a class that implements C directly or indirectly.

This explains why `MY_STATIC` might be null, but it does not explain the NPE itself, since there is no operation on `val` in the code.  
Let's decode the class file:
```
public static final java.lang.Object MY_STATIC;  
public default java.lang.Object getMyStatic();
  Code:
     0: getstatic     #1                  // Field MY_STATIC:Ljava/lang/Object;
     3: areturn
static {};
  Code:
     0: new           #7                  // class com/mycompany/myapp/MyInterface$1
     3: dup
     4: invokespecial #9                  // Method com/mycompany/myapp/MyInterface$1."<init>":()V
     7: putstatic     #1                  // Field MY_STATIC:Ljava/lang/Object;
    10: return
```

The class looks as expected. Let's attach the debugger from Xcode and catch the crash:
```
  Main`[J]com.mycompany.myapp.MyInterface.getMyStatic()Ljava/lang/Object;:
      0x10173de30 <+0>:  adrp   x8, 8751
      0x10173de34 <+4>:  add    x8, x8, #0x90
      0x10173de38 <+8>:  ldr    x8, [x8]
  ->  0x10173de3c <+12>: ldr    x0, [x8, #0xa0] # EXC_BAD_ACCESS
      0x10173de40 <+16>: ret 
```

And it looks like the following LLVM IR code is being inlined:
```
define weak %Object* @"[J]com.mycompany.myapp.MyInterface.getMyStatic()Ljava/lang/Object;"(%Env* %p0, %Object* %p1) nounwind noinline optsize {
label0:
  %r0 = alloca %Object*
  %$r1 = alloca %Object*
  store %Object* %p1, %Object** %r0, !dbg !{i32 16, i32 0, !7, null}
  %t0 = call %Object* @"[j]com.mycompany.myapp.MyInterface.MY_STATIC(Ljava/lang/Object;)[get]"(%Env* %p0), !dbg !{i32 16, i32 0, !7, null}
  store %Object* %t0, %Object** %$r1, !dbg !{i32 16, i32 0, !7, null}
  %t1 = load %Object** %$r1, !dbg !{i32 16, i32 0, !7, null}
  ret %Object* %t1, !dbg !{i32 16, i32 0, !7, null}
}
define private %Object* @"[j]com.mycompany.myapp.MyInterface.MY_STATIC(Ljava/lang/Object;)[get]"(%Env* %p0) nounwind alwaysinline optsize {
label0:
    %t0 = call i8** @"[j]com.mycompany.myapp.MyInterface[info]"()
    %t1 = load i8** %t0
    %t2 = bitcast i8* %t1 to i8*
    %t3 = getelementptr i8* %t2, i32 ptrtoint (%Object** getelementptr (%ClassType* null, i32 0, i32 1, i32 0, i32 1) to i32)
    %t4 = bitcast i8* %t3 to %Object**
    %t5 = load %Object** %t4
    ret %Object* %t5
}
```

It seems to access `ClassInfoStruct` (via `"[info]"`), load the first pointer that is expected to be the `Clazz` structure for `MyInterface`, and then navigate to its end. 
Right after the `Clazz` structure, the static field `MY_STATIC` is expected to be located.
It then tries to read it and crashes with `EXC_BAD_ACCESS`, which means the calculated address is invalid.  
This happens only if the `ClassInfoStruct.clazz` field is not initialized (`null`), which is a valid state when the class has not been loaded yet. 

To validate the idea, let's load it but not initialize it:
```java
  System.out.println(MyInterface.class);
  MyInterface obj = new MyImpl();
  Object val = obj.getMyStatic();
  System.out.println(val);
```

And here it prints `NULL`! `MY_STATIC` is still not initialized, but at least there is no crash.

## The fix 
Adapt to `Java Virtual Machine Specification, 5.5`, and initialize any interface that has a default method: 
- when a class is loaded, recursively check all of its interfaces;
- if an interface has a default method, initialize it;
- as an optimization, if an interface has already been visited, mark it and skip it next time.

Sadly, we cannot mark classes as containing interfaces with default methods at compile time and check only those at runtime. 
At the moment a class is being compiled, its dependencies are not yet compiled either, and they are scheduled only after the current class compilation finishes.

Fix delivered as [PR#39](https://github.com/robovmx/robovmx/pull/39)
