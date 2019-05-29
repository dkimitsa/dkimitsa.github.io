---
layout: post
title: "CocoaTouch: tonns of wrong enum marshalers"
tags: [bro-gen, binding]
---
Once [Issue #373 (Wrong data type of MIDINotificationMessageID enum)](https://github.com/MobiVM/robovm/issues/373) was opened it is clear that cocoa-touch had to be revised for enum marshalers case.   
This issue wasn't exposed till [#373](https://github.com/MobiVM/robovm/issues/373) and was around for years but anyway it is just metter of time till someone finds another one.

# What's wrong
Marshalers are compilation time annotations that specifies RoboVM compiler the way how to convert native/objc object to java side and back. In case of enums it is usualy specifies size of integer used by enum type in native/objc (e.g. signed/unsigned char/short/int/long).  
Using wrong size/sign integer type marshalers leads to data loss (when wrong value read from memory) and potentionaly memory corruption on write attempt.  
* using enum type as member of structure ([#373](https://github.com/MobiVM/robovm/issues/373))
* accessing enum type value by pointer

# Why it was in general working 
<!-- more -->
In most case value of enum passed in one of general purpose register of CPU (for example rax/eax) even if it was represented by single byte.   
So this was not causing issue when 1 byte was written (on function returning enum type) to 64 bit register and 32 bit value then was read from it on using.

# Scenarios to face issues 
Bellow are wrong usage cases of marshalers fixed in cocoa touch bindings:

## Case1: MachineSizeInt marhallers were used
One of the popular mistake and root case of [#373](https://github.com/MobiVM/robovm/issues/373).
Root case -- it was wrongly manually configured in yaml files. 
It tells to use machine arch dependent integer marshaler (e.g. NSInteger) which is 32bit wide on 32bit systems and 64 bit wide otherwise. It becomes a problem when enum is fixed width integer for ex. uint32_t.

## Case2: bro-gen was not generating marshalers but only for MachineSizeInt ones
That was a case, as long as there was no type specified for enum or enum were typed to signed/unsigned 32bit integer -- everything worked. But there are some enum that uses storage type as byte/short/long.

## Case3: bro-gen was wrongly generating enum values for unsigned enums
Even if attached marshaler was correct -- java side values often got negative constants. For example for 32 bit unsigned storage type and enum entry value of 4294967295 bro-gen generated constant of -1. As result such constant were failing to marshal on Java side as Java enum doesn't contain entry with such value.

# The clean-up
* First thing bro-gen was updated to generate proper marshaler automaticaly. 
* Second -- yaml files were inspected and all not required marshaler directives (ones that bro-gen can recognize) were removed.   
* Third -- most iOS SDK migrate to defining enums with `NS_ENUM` which simplified configuration of ones in yaml. Yamls were cleaned up from not required any more configuration data. 
* As result all bindings were re-generated. 

# The code
* `Cocoa touch` bindings were delivered as [PR377](https://github.com/MobiVM/robovm/pull/377)  
* Updated `bro-gen` pushed into [dkimitsa/robovm-bro-gen](https://github.com/dkimitsa/robovm-bro-gen)

# When manual marshaler is needed 
There are cases when marshaler is needed to be manually specified in yaml file. The case is simple:
* enum is used to specify constants;
* constants are used with completely stand-alone types.

Example
```c
enum {
  CONST_A,
  CONST_B
};

typedef unsigned char FooEnum;
@interface Foo
+update:(FooEnum)foo;
@end
```

in yaml file this can be done two ways: specifying marshalers or specifying type of enum and allow bro-gen to resolve marshaler:  
```yaml
enums:
     EnumA: {marshaler: marshaler: ValuedEnum.AsUnsignedIntMarshaler}
     EnumB: {type: marshaler: uint8_t}
``` 
