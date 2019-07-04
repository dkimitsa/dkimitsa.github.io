---
layout: post
title: 'bro-gen: fixes + argument tuples'
tags: ['bro-gen', whatsnew, binding]
---
Another set of bro-gen updates it get during everyday usage. Whats new:  
- arguments tuple 
- better support for @Deprecated annotation
- fixed visibility of default constructor  
## arguments_tuple
<!-- more -->
Its used with constructor method definition in yaml. When used -- it will replace all arguments with with signle tuple class argument and will put all arguments as filds of tuple. It introduce to handle case when two(or more) different objc init methods result in java constructors with same signature.
Example, obj-c class has following initializers:
```objc
@interface Foo
-(id)initWithX:(int)x Y:(int)y;
-(id)initWithWidth:(int)width Height:(int)height;
@end
```
will produce java constructors with same signature (not compilable code):
```java
class Foo {
   Foo(int x, int y) {}
   Foo(int width, int height) {}
}
```
Can be workarounded with `arguments_tuple` (yaml):
```yaml
classes:
  foo:
    methods:
      `-initWithWidth:Height:`:
        name: init
        arguments_tuple: ArgsWidthHeight
```

Results following bindings:
```java
    /** argument tuple for constructor bellow */
    public static class ArgsWidthHeight {
       public final int width;
       public final int height;
       public ArgsYData(int width, int height) {
          this.width = width;
          this.height = height;
       }
    }
    @Method(selector = "-initWithWidth:Height:")
    public foo(ArgsWidthHeight tuple) { super((SkipInit) null); initObject(init(tuple.width, tuple.height)); }

```

## Deprecation annotation 
Added support for objective c deprecation attributes:
- __deprecated_msg
- __deprecated

As result more methods get deprecated in CocoaTouch/Robopods!  
Also bro-gen was fixed to put `@Deprecated` annotation to proper location (was in javadoc). This produced bunch of cosmetic changes in CocoaTouch source code. Was before:  
```java
/**
 * @since Available in iOS 2.0 and later.
 * @deprecated Deprecated in iOS 9.0. use CNAuthorizationStatus
 */
@Deprecated
/*</javadoc>*/
/*<annotations>*/@Marshaler(ValuedEnum.AsMachineSizedSIntMarshaler.class)/*</annotations>*/
public enum /*<name>*/ABAuthorizationStatus/*</name>*/ implements ValuedEnum {
```
now:
```java
/**
 * @since Available in iOS 2.0 and later.
 * @deprecated Deprecated in iOS 9.0. use CNAuthorizationStatus
 */
/*</javadoc>*/
/*<annotations>*/@Marshaler(ValuedEnum.AsMachineSizedSIntMarshaler.class) @Deprecated/*</annotations>*/
public enum /*<name>*/ABAuthorizationStatus/*</name>*/ implements ValuedEnum {
```

## fixed visibility of default constructor

bro-gen always generate default constructor. it is required to be able instantiate Custom object from native side (e.g. its a way how ApplicationDelegate gets instantiated). 
but bro-gen was generating constructors as public just if `-init` method was present in class/super class. Once this condition fixed -- it discover lot of classes in CocoaTouch that had methods marked as `exclude`  but generated as public. In most cases proper fix was to remove `exclude` flag from yaml file.

## Source code 
Recent bro-gen available in [dkimitsa/robovm-bro-gen](https://github.com/dkimitsa/robovm-bro-gen)  
Updated bindings were delivered as part of [fixed enum marshallers PR377](https://github.com/MobiVM/robovm/pull/377) to minimize number of PRs.
