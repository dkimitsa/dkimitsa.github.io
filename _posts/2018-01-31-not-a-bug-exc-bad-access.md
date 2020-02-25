---
layout: post
title: 'not a bug: EXC_BAD_ACCESS if object created with method whose name begins with “alloc”, “new”, “copy”, or “mutableCopy”'
tags: [fix, target-framework, arc, gc]
---
**UPDATED** with workaround if "new" prefix is still required.

Native ObjectiveC/Swift code that uses shared code [RoboVM code as Framework]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %}) could ends in following:
![]({{ "/assets/2018/01/31/exc-bad-access.png"}})

Sometimes it happens randomly but here is how to make it upon-request:   
<!-- more -->
Lets modify `MyFrameworkImpl` of project created from Framework template as bellow:
```java
public class MyFrameworkImpl extends NSObject implements Api.MyFramework {
    @Override public void sayHello() {
        System.out.println("Hello world from RoboVM framework");

        /*added*/ for (int i = 0; i < 100; i++)
        /*added*/    System.gc();
    }
}
```

It is clear that crash happens due GC but hey, it should be retained at iOS native side. But it is not due expected ARC behavior: all problem in method name. Its time to [RTFM Memory Management Policy](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/MemoryMgmt/Articles/mmRules.html):
> *You own any object you create*   
> You create an object using a method whose name begins with “alloc”, “new”, “copy”, or “mutableCopy” (for example, alloc, newObject, or mutableCopy).

## The fix - do not let ARC to think that it owns the object
Just don't name methods that create java object instances with prefixes listed above. Prefix `create` works like a charm. Already updated [framework template](https://github.com/MobiVM/robovm/pull/260).

## Root case
Lets check how ARC acts (I traced it with modified RoboVM runtime but will show here just steps), evaluate following code snippet:
```swift
class ViewController: UIViewController {
    var calculator : Calculator!;

    @IBAction func test(_ sender: Any) {
        MyFrameworkInstance().sayHello();
        calculator.add(123);
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        calculator = MyFrameworkInstance().newCalculator()
        // Do any additional setup after loading the view, typically from a nib.
    }
}
```

* `MyFrameworkInstance().newCalculator()` returns object create at Java side with retainCount == 1;
* at this moment ARC call `retain` to the object to retain intermediate result, retainCount == 2 at this moment;
* then assigns value to calculator field, and doesn't retain. As it consider it owns it (check `You own any object you create` above);
* after assigning it call `release` to the object to release intermediate result, retainCount == 1 at this moment.

From ARC perspective it absolutely valid case as it consider object to be owned by code (not externally).

## But not valid case for RoboVM
For any native object create in RoboVM two objects exists:
* Java object;
* And native object.

And these object are connected to each other. If any side got released and destroyed -- counterpart shall follow. If Java object is kept by native object (our case as Calculator) following logic follows ([check code](https://github.com/MobiVM/robovm/blob/aa4d55d5cc72e0a58ab597d5afd4d5c404308e4a/compiler/objc/src/main/java/org/robovm/objc/ObjCObject.java#L477)):
- on each native `retain()` call object's reference is kept in RoboVM set of objects;
- on each native `release()` it checks `retainCount()` and if it goes to 1 it removes object's reference from RoboVM set of objects (now it is subject for GC).

Once `retainCount()` is 1 it means that there is no object at Native side keeping reference to it and only Java counterpart is retaining it. And once Java counterpart is GC-ed it also `release()` the native part causing it destruction.

## Making a case valid for RoboVM
The best approach would be not use these prefixes especially if developer pure Java/Android and has not knowledge in not used now manual reference counting at iOS side. But if it is required to have the name with one of the prefix listed above object has to be extra retained on Java side. Consider following snippet:
```java
@Override
public Api.Calculator newCalculator() {
    CalculatorImpl calc = new CalculatorImpl();
    calc.retain();
    return calc;
}
```

It will return object with retainCount() set to 2 which will not lead to EXC_BAD_ACCESS. And object will be released once java counterpart is released.

## Bottom line
In this bright example ARC considers that code owns the object and retained it. RoboVM saw retain count decreased to 1 and considers that it not needed anymore and destroyed it GC cycle.
Object returned by a method whose name begins with “alloc”, “new”, “copy”, or “mutableCopy” (for example, alloc, newObject, or mutableCopy) will not be maintained by ARC. RoboVM developer shall consider this fact.
