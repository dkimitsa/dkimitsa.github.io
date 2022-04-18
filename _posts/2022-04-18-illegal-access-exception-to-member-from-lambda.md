---
layout: post
title: 'Fix: java.lang.IllegalAccessError: Attempt to access protected method from Lambda'
tags: [fix, debugger, ios15, dyld]
---
Issue mentioned in report [#641](https://github.com/MobiVM/robovm/issues/641)
Code is quite simple to reproduce:
```java
package a;
public class A {
    protected void foo(){};
}

package b;
import a.A;
public class B extends A {
    public void test() {
        Runnable r = this::foo;
        r.run();
    }
}
```

## Root case
<!-- more -->
Is a way how RoboVM generate lambda class for this case, it looks as bellow:
```java
package b;

public class B$lambda$1 implements Runnable{
    private final B $arg1;

    public B$lambda$1(B $arg1) {
        this.$arg1 = $arg1;
    }

    @Override
    public void run() {
        $arg1.foo();
    }
}

// and caller class 
public class B extends A {
    public void test() {
       Runnable r = new B$lambda$1(this); // was Runnable r = this::foo;
       r.run();
    }
}
```

and generated `$arg1.foo();` will produce `IllegalAccessError` as `foo() has protected access in A`.

# The fix
To work around this -- lets make same trick as its done to provide access to private fields from lambdas. Lets create a trampoline method in `B` that will call method handle for this lambda and generate lambda code to call this trampoline.
Method handle has signature:
> implMethod =  "InvokeVirtual(<a.A: void foo()>)"

This to be turned into following trampoline:
```java
package b;

public class B$lambda$1 implements Runnable{
    private final B $arg1;

    public B$lambda$1(B $arg1) {
        this.$arg1 = $arg1;
    }


    @Override
    public void run() {
        B.trampoline$lambda$1($arg);
    }
}

// and caller class 
public class B extends A {
    public void test() {
        Runnable r = new B_lambda_1(this); // was Runnable r = this::foo;
        r.run();
    }

    private static void trampoline$lambda$1(A arg) {
        arg.foo();
    }
}
```

But trampoline will not work this was as protected `foo` method of receiver type `A` is not accessible from package `B`.

Lambda is created by `LambdaMetafactory` and `receiver` object is passed as first parameter of `invokeType` Callsite's signature. And it will match calling class, not actual class where member is located.
The hack is: when building trampoline, use first argument type from `invokeType` parameter list and rest from MethodHandle provided implementation.
This sample trampoline with all fixes:
```java
  private static void trampoline$lambda$1(B arg) {
    arg.foo();
  }
```

# Code
The fix was proposed as [PR645](https://github.com/MobiVM/robovm/pull/645)
