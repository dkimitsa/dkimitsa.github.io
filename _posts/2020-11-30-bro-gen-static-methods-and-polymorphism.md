---
layout: post
title: 'bro-gen: static methods and polymorphism'
tags: ['bro-gen']
---
There is one binding trick related to inherited class methods/properties in objc code. Polymorphism is not available for static methods in Java and compiler resolves target for static call during compilation time. Following obj-c code and Java binding demostrate the issue:  
```objc
@interface Abstract : NSObject
+(void) test;
@end

@interface TestObject : Abstract
@end
```  
Corresponding binding:
```java
public class Abstract extends NSObject {
    public Abstract() {}
    @Method(selector = "test")
    public static native void test();
}

public class TestObject extends Abstract {
    public TestObject() {}
}
```

Java call `TestObject.test()` will fail while obj-c call `[TestObject test]` is not. Root case -- static method invocation is resolved compilation time and `test()` method is present in `Abstract`. This produces `Abstract.test()` call that corresponds to obj-c call `[Abstract test]`. 

# The fix
Is to copy all static methods/properties from super-classes/protocols into target class.  Bro-gen script [was updated](https://github.com/dkimitsa/robovm-bro-gen/commit/414c9fb8c3b9154c421f2b0fe127b2616e12f116) to do this automatically. 
