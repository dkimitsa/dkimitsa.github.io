---
layout: post
title: "binding: few lessons learned"
tags: [binding]
---
While testing binding of `Network.framework` there were several discoveries:  
1. out of box binding to global values or bridge functions from linked static library will not work;
2. RoboVM is not able to compile global value that returns a obj-c block;
3. Don't use method name `copy` when binding function. It will crash during compilation.

Details and workarounds bellow.  
<!-- more -->

## 1. Values and Functions from statically linked library
Following native code, being compiled in static library:
```objc
const float testFloatGlobal = 123.456f;
void testFloatFunction(float v) {
    NSLog(@"testFloatFunction with %f", v);
}
```

and bind into following class:
```java
@Library(Library.INTERNAL)
public class ExportSymbTest extends NSObject {
    @GlobalValue(symbol = "testFloatGlobal")
    static native float testFloatGlobal();

    @Bridge(symbol = "testFloatFunction")
    static native void testFloatFunction(float v);
}

```

will fail with runtime exception:
> java.lang.UnsatisfiedLinkError: Failed to resolve symbol 'testFloatGlobal' for method static native float com.mycompany.myapp.ExportSymbTest.testFloatGlobal() with @GlobalValue annotation @org.robovm.rt.bro.annotation.GlobalValue(symbol=testFloatGlobal, optional=false, dereference=true) in library @org.robovm.rt.bro.annotation.Library(value=__internal__)
	at org.robovm.rt.bro.Runtime.resolveGlobalValue(Runtime.java:233)

and  
> java.lang.UnsatisfiedLinkError: Failed to resolve native function 'testFloatFunction' for method static native void com.mycompany.myapp.ExportSymbTest.testFloatFunction(float) with @Bridge annotation @org.robovm.rt.bro.annotation.Bridge(symbol=testFloatFunction, dynamic=false, optional=false) in library @org.robovm.rt.bro.annotation.Library(value=__internal__)
    	at org.robovm.rt.bro.Runtime.resolveBridge(Runtime.java:210)


**Reason for this**: is the way RoboVM resolves these symbols. RoboVM doesn't create static reference to this symbols and uses `dlsym` to locate it. All these symbols are not get exported to dynamic symbol table and runtime is not able to locate these. So crashes.

**Solution**: Best case is to automatically find all such cases and add symbols to the exported list (one day might do this) but today we have to add them manually to the `robovm.xml`:  
```xml
<exportedSymbols>
    <symbol>testFloatFunction</symbol>
    <symbol>testFloatGlobal</symbol>
</exportedSymbols>
```

## 2. RoboVM is not able to compile global value that returns a obj-c block  
Simple case, following binding will crash compilation:  
```java
@Library(Library.INTERNAL)
public class ExportSymbTest extends NSObject {
    @GlobalValue(symbol = "testBlockGlobal")
    static native @Block VoidBlock1<Float> testBlockGlobal();
}
```

With exception:
> [ERROR] Couldn't compile app
java.lang.IllegalArgumentException: No @Marshaler found for return type of @GlobalValue method <com.mycompany.myapp.ExportSymbTest: org.robovm.objc.block.VoidBlock1 testBlockGlobal()>
	at org.robovm.compiler.MarshalerLookup.findMarshalerMethod(MarshalerLookup.java:169)

**Reason for this**: marshaling blocks was not done as it was not required till a moment I believe. But there are few constants in `Network.framework` that are accessible this way. Fix provided with [commit](https://github.com/MobiVM/robovm/pull/335/commits/23aefba1b7b08403237e76d83abddd8929989bb2)

## 3. Don't use method name `copy` when binding function
Check following code:
```java
@Library(Library.INTERNAL)
public class ExportSymbTest extends NSObject {
    @Bridge(symbol = "test_copy_symb")
    public native ExportSymbTest copy();
}
```

It crashes compilation with:  
>[ERROR] Couldn't compile app
java.lang.IllegalArgumentException: @Bridge annotated method <com.mycompany.myapp.ExportSymbTest: org.robovm.apple.foundation.NSObject copy()> must be native
	at org.robovm.compiler.BridgeMethodCompiler.validateBridgeMethod(BridgeMethodCompiler.java:87)

But there is no method `NSObject copy()` in `ExportSymbTest`, but it is in NSObject itself.

**Reason for this**: Java compiler creates synthetic bridges to handle effects of type erasure,  details [by link](https://docs.oracle.com/javase/tutorial/java/generics/bridgeMethods.html). And the problem is that bridge receives all annotations of method bridge is built to. In our case this is `@Bridge` and RoboVM compiler fails on it during validation.

Decompiled .class file has both these methods:
```
$ javap -c ExportSymbTest.class  
Compiled from "ExportSymbTest.java"                                                                           
public class com.mycompany.myapp.ExportSymbTest extends org.robovm.apple.foundation.NSObject {                
  public com.mycompany.myapp.ExportSymbTest();                                                                
    flags: ACC_PUBLIC, ACC_NATIVE
    Code:                                                                                                     
       0: aload_0                                                                                             
       1: invokespecial #1                  // Method org/robovm/apple/foundation/NSObject."<init>":()V       
       4: return                                                                                              

  public native com.mycompany.myapp.ExportSymbTest copy();                                                    

  public org.robovm.apple.foundation.NSObject copy();                                                         
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:                                                                                                     
       0: aload_0                                                                                             
       1: invokevirtual #2                  // Method copy:()Lcom/mycompany/myapp/ExportSymbTest;             
       4: areturn                                                                                             
}                                                                                                             
```

There is no out of box solution for it, probably after investigation it would be enough to drop all `ACC_BRIDGE` methods. But today just try not have type erasure case when binding functions/globals.
