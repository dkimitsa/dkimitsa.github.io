---
layout: post
title: "iOS13 PostFix #1: Generic class arguments and @Block parameters"
tags: ["fix", "compiler", "postfix"]
---
iOS13 bindings [are complete]({{ site.baseurl }}{% post_url 2019-09-18-ios-13-binding %}) but CocoaTouch related issues keep arriving, part of them about compiler not able to compile one or other class.  
Most of these issues can be detected just by compiling entire CocoaTouch library as simple smock test to find out outstanding issues. This can be done by force linking its class path with template like this:
```xml
<forceLinkClasses>
    <pattern>org.robovm.apple.**</pattern>
</forceLinkClasses>
```

This post start series of fixes discovered during compilation of CocoaTouch library.  
# PostFix #1: Generic class arguments and @Block parameters
Other postfixes:  
* [PostFix #2: Adding support for non-static @Bridge method in enums classes]({{ site.baseurl }}{% post_url 2019-10-20-cocoa13-postfix2-non-static-bridge-methods-in-enum %})
* [PostFix #3: Support for @Block member in structs]({{ site.baseurl }}{% post_url 2019-10-21-cocoa13-postfix3-blocks-in-structs %})
* [PostFix #4: Compilation failed on @Bridge annotate covariant return synthetic method]({{ site.baseurl }}{% post_url 2019-10-22-cocoa13-postfix4-covariant-return-bridge %})
* [PostFix #5: Support for Struct.offsetOf in structs]({{ site.baseurl }}{% post_url 2019-11-17-cocoa13-postfix5-offsetof-in-structs %})
* [PostFix #6: Fixes to Network framework bindings]({{ site.baseurl }}{% post_url 2019-12-18-cocoa13-postfix6-network-framework-bindings %})

<!-- more -->
##  What is wrong 
Issue was detected while compility [NSOrderedCollectionDifference.getDifferenceByTransformingChanges](https://github.com/MobiVM/robovm/blob/e48e3adbf9d5c14023c1ec4f4f3e413191dc1245/compiler/cocoatouch/src/main/java/org/robovm/apple/foundation/NSOrderedCollectionDifference.java#L88).  
It failed with following exception:  
> org.robovm.compiler.CompilerException: Unresolved type variable T in parameter 1 of @Block method

### Setups 
All tests were perfromed by modifying [ObjCBlockPluginTest.java](https://github.com/MobiVM/robovm/blob/83e43dfc72213fcde6cb46e996c0dab164ce64b7/compiler/compiler/src/test/java/org/robovm/compiler/plugin/objc/ObjCBlockPluginTest.java) as it provides quick run experience. For test following block interface will be used:   
```java
public interface Block<R, A, B> {
    R run(A a, B b);
}
```

### Case 1, reproducing CocoaTouch issue
Attempt to use following generic class with block argument...  
```java
public static class BlockTest<T> {
    public native void foo(Block<T, Integer, Float> b);
}
```

Cause exception:  
> org.robovm.compiler.CompilerException: Unresolved type variable T in return type of @Block method 

(Strange) By the way, such cace is expected in UT [`testResolveTargetMethodSignatureGenericWithUnresolvedIndirectTypeVariable`](https://github.com/MobiVM/robovm/blob/83e43dfc72213fcde6cb46e996c0dab164ce64b7/compiler/compiler/src/test/java/org/robovm/compiler/plugin/objc/ObjCBlockPluginTest.java#L214).

But...

### Case 2
Changing a generic class argument name (ONLY!) fixes the exception from case 1. 
```java
public static class BlockTest<R> {
    public native void foo(Block<R, Integer, Float> b);
}
```

### Case 3 -- more bugs
Swaping type arguments... 
```java
    public static class BlockTest<R, A, B> {
        public native void foo(Block<R, B, A> b);
    }
```  
causes recursion loop and stack overflow exception in result:  

>java.lang.StackOverflowError
>  at org.robovm.compiler.plugin.objc.ObjCBlockPlugin.resolveMethodType(ObjCBlockPlugin.java:903)
>  at org.robovm.compiler.plugin.objc.ObjCBlockPlugin.resolveMethodType(ObjCBlockPlugin.java:920)


### Case 4 -- more bugs2
Naming class arguments same to Block arguments casuses in wrong type resolution:
```java
public static class BlockTest<R, B extends Integer> {
    public native void foo(Block<R, B, Float> b);
}
```
results in following wrong block:
```java
public interface Block<R, A, B> {
    java.lang.Object run(java.lang.Float a, java.lang.Float b);
}
```
 
## Root case 
Root case is [`ObjCBlockPlugin.resolveMethodType`](https://github.com/MobiVM/robovm/blob/83e43dfc72213fcde6cb46e996c0dab164ce64b7/compiler/compiler/src/main/java/org/robovm/compiler/plugin/objc/ObjCBlockPlugin.java#L899) method. It take folowing arguments for class `BlockTest` (Case 1) and `Block` (Setups):
```
Type t:                            R
Type[] resolvedArgs:               [T extends Object (TypeVariable), Integer, FLoat]
TypeVariable<?>[] typeParameters:  [R, A, B] 
```  

Problem here is that it tries to resolve all TypeVariable types using following scheme:
- find parameter index in `typeParameters` by name -- 0 in this case;
- get its value from type parameters: `t = typeParameters[index]` - T in this case;
- resolve this value by recursively calling `resolveMethodType(t)` - fill fail as there is no T in `typeParameters:` [R, A, B]

And problems here:  
- it can't resolve any argument parameter, other than in typeParameters (but its wrong by itself);
- even if name matches -- it will pick argument by typeParameters.indexOf(t.name) which will cause resolution of not related type, and wrong result (case 4);
- several arguments name matche but their position doesn't match ones in typeParameter this will cause recursion and StackOverflow exception;

## The fix
[PR419](https://github.com/MobiVM/robovm/pull/419)

Idea of fix -- is not have any TypeVariable in resovedArgs. For this all resolved arguments are initialy being resolved to their java bounds, as result parameter types `resolveMethodType` will not recursively resolve TypeVariable. But insted if TypeVariable is passed it expects it to be one of typeParameters and return already resoved value from resolvedArgs by it index. Otherwise CompilerException will be thrown.
