---
layout: post
title: 'Framework improved -- exposing ObjectiveC classes to natively create instances'
tags: ['target-framework', 'tutorial']
date: 2020-02-21 16:00:00 +02:00
---
[Previous improvement]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %}) simplified way Framework target was created right out the box without need to write native code. But it didn't allow creating objects native way and workaround was to use single point singleton and consider objects as protocols.
RoboVM creates objects runtime using objc/runtime API so there is no information available during linkage phase. And a try to use it will cause symbol not found exception like this:
>Undefined symbols for architecture x86_64:
>   "_OBJC_CLASS_$_RoboClass", referenced from:
>       objc-class-ref in ViewController.o

The goal of this Improvement:
* hide any need to initialize JVM from user;
* allow natively creating and using of java classes as native.

In general it should be enough:
- drop framework in Xcode project;
- instantiate and use classes like bellow:
```
let demo = MyFrameworkDemo(text: "demo")!
print(demo.roboVmVersion()!)
// or
MyFrameworkDemo *demo = [[MyFrameworkDemo alloc] initWithText:@"demo"];
NSLog(@"%@", [demo roboVmVersion]);
```

[(skip long read and start creating framework)](#end-of-long-read)

# Step one. OBJC_CLASS_$_${CLASSNAME} structure
<!-- more -->
`OBJC_CLASS_$_${CLASSNAME}` represent structure of class, one that is being returned by `[MyClass class]`. For ObjectiveC v2 its definition is not available in SDK header files but it can be found in Apple/Objc4 [objc-runtime-new.h](https://opensource.apple.com/source/objc4/objc4-437/runtime/objc-runtime-new.h) and it looks as bellow:
```c
typedef struct class_t {
    struct class_t *isa;
    struct class_t *superclass;
    Cache cache;
    IMP *vtable;
    class_rw_t *data;
} class_t;
```

In general it points to 5 different structures that describes superclass, methods, members and so on. Compiler is responsible to generate and export all these structure. So proper way would be to teach RoboVM compiler generating these.

But its to much to be done. RoboVM creates backing ObjectiveC classes runtime during loading matching Java class.

Solution is to hack:
* compiler will generate `OBJC_CLASS_$_${CLASSNAME}` for each custom java class and framework will expose it. This allows to write native initialization of object and binary binary will be linked without `Undefined symbols` error;
* runtime will create backing ObjC class using objc/runtime as usual;
* and *THE TRICK*: code will copy `class_t` created using objc/runtime api into `OBJC_CLASS_$_${CLASSNAME}` structure and will make it valid.

The moment here -- is to fill `OBJC_CLASS_$_${CLASSNAME}` with valid information before class is used on native side. In other word -- corresponding Java class has to be loaded before its ObjectiveC counter part is used.

# Step two. Starting JVM automatically and preloading ObjC classes
[Previous improvement]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %}) framework support introduced api that allows to initiate JVM and preload java class (singleton of framework) manually:
```c
//
// static function that returns instance of Framework's main class. On first access it also instantiate RoboVM
//
static MyFramework* MyFrameworkInstance() {
    extern MyFramework* rvmInstantiateFramework(const char *className);
    return rvmInstantiateFramework("com.mycompany.myframework.MyFrameworkImpl");
}
```

And in general all classes might be preloaded by `com.mycompany.myframework.MyFrameworkImpl` java class. But it is not fluent in 2020.
RoboVM framework target support was changed to do this automatically when Application loads framework(magic!). Following was done:
* compiler makes a list of all classes annotated with @CustomClass annotation and available in class path (and not dropped by tree shaker);
* frameworksupport.m extended with `__attribute__((constructor))` module initializer function that is triggered, when framework is loaded, details [at Apple](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPFrameworks/Tasks/InitializingFrameworks.html);
* module inializer starts JVM (so you don't have to bother about) and loads all @CustomClass classes exposed by RoboVM compiler.

At this moment java classes can be natively used in ObjectiveC/Swift native code.

### end of long read

# Tutorial: Implementing Framework new way

### Extended from NSObject, annotate with @CustomClass("ClassName")
Exposed classes HAVE to be extended from NSObject (or another class that extended from it) and annotated with @CustomClass annotations:
```java
@CustomClass("MyFrameworkDemo")
public class MyFrameworkDemo extends NSObject {
// ...
}
```

`@CustomClass` annotation should specify a name this class will be used in native part (it might don't match Java one but it is better to keep these same).

### Annotate constructors/methods with @Method annotation
`@Method` annotation connects a method with ObjectiveC selector it going to bind to. +/- prefix of selector should not be specified.
Constructor have to be bind to `-init` initializer:
```java
    /**
     * mapping constructor to -(void)init selector
     * IMPORTANT: no minus sign !
     */
    @Method(selector = "init")
    public MyFrameworkDemo() {
        System.out.println("Default constructor was called!");
    }

    /**
     * secondary constructor with parameter.
     * mapped to -(void)initWithText:(NSString*)test selector
     * IMPORTANT: no minus sign in @Method annotation
     */
    @Method(selector = "initWithText:")
    public MyFrameworkDemo(String text) {
        System.out.println("Secondary constructor with " + text + " called!");
    }
```

### Writing `headers/MyFramework.h`
This file is specifies API that native code will see and use. It content should match class names specified in `@CustomClass` and selectors should match ones in `@Method(selector = )` annotations:
```c
//
// MyFrameworkDemo class with basic API demonstration
//
@interface MyFrameworkDemo
-(id)init;
-(id)initWithText:(NSString*)text;
+(void)hello;
-(NSString*)roboVmVersion;
-(void)installSignals:(void(^)(void))installer;
@end
```

### Tell treeshaker what classes to keep
Framework target uses aggressive tree shaker by default which will drop all not referenced stuff, so configure `robovm.xml` to have exposed classes force linked:
```xml
    <!-- Force link all classes in the SDK packages. -->
    <forceLinkClasses>
        <pattern>com.mycompany.myframework.**</pattern>
    </forceLinkClasses>
```

### Natively use your Java code in ObjectiveC/Swift application
At this point project should be compiled into framework. Included in Xcode project and natively used like bellow:
```swift
import MyFramework
// ...
let demo = MyFrameworkDemo(text: "demo")!
print(demo.roboVmVersion()!)
let calc = Calculator(value: 1000)!
calc.add(12)
print(calc.result())
```

### Framework template
Framework template was updated to these changes and its good idea to start with it. Available in Idea plugin.

# Code
Code was delivered as [PR457](https://github.com/MobiVM/robovm/pull/457)

