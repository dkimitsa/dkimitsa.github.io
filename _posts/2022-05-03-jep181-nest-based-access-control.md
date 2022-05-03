---
layout: post
title: 'JEP 181: Nest-Based Access Control'
tags: [compiler, jep, java11]
---
RoboVM stopped to be Java7+ runtime (Android 4.4 to be exact). However it doesn't stop users to compile it against more recent Java like java 11.  
It works with some amount of constraints but sometime changes are breaking. Like [JEP181](https://openjdk.java.net/jeps/181) that causes java8 scenarios doesn't work anymore with RoboVM ([issue #852](https://github.com/MobiVM/robovm/issues/652)).  
Long story short: with JEP181 changes compiler doesn't generate accessors to private fields for nested classes. And RoboVM truly produces `IllegalAccessError: Attempt to access private method/variable`.  
Here is a sample code to reproduce:
<!-- more -->
```java
public class Test {
    private int a;
    public void foo() {
        Runnable r = new Runnable() {
            @Override
            public void run() {
                System.out.println(a);
            }
        };
        r.run();
    }
}
```

Instead, `Java11+` compiler generates `NestHost` and `NestMembers` attributes for classes:
- `NestHost` specifies host class for nested one;
- `NestMembers` of host class specifies nested members.

And up to [JEP181](https://openjdk.java.net/jeps/181):
```
A field or method R is accessible to a class or interface D if 
and only if any of the following conditions are true:  
  * ...  
  *  R is private and is declared in a different class or
     interface C, and C and D, are nestmates.
For types C and D to be nestmates they must have the same nest host. 
A type C claims to be a member of the nest hosted by D, if it lists D 
in its NestHost attribute. The membership is validated if D also 
lists C in its NestMembers attribute. D is implicitly a member of 
the nest that it hosts.
A class with no NestHost or NestMembers attribute, implicitly forms 
a nest with itself as the nest host, and sole nest member.
```
Things sound like [cpp friend classes](https://docs.microsoft.com/en-us/cpp/cpp/friend-cpp?view=msvc-170) now...

## Compiler time fix
To support `JEP181` during compilation it is enough to check `NestHost` and `NestMembers` attributes as described.   

## Limitation
Beside static case (compilation time) it shall work with runtime/regression as well. This to be done as part2, probably in scope of [Libcore10](https://github.com/robovmx/robovmx/tree/experiment/2-libcore-10) experiment where corresponding API is exposed.  

## Code
- Changes proposed to [Soot #3](https://github.com/MobiVM/soot/pull/3) to expose `NestHost` and `NestMembers` attributes.   
- RoboVM changes proposed as [PR653](https://github.com/MobiVM/robovm/pull/653).   
- Both there already available in [RoboVMx/Libcore10](https://github.com/robovmx/robovmx/tree/experiment/2-libcore-10) branch and subject for build3. 

Happy coding and please open an issue if any.
