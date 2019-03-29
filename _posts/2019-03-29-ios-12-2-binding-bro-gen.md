---
layout: post
title: "iOS 12.2 bindings, what's new in bro-gen"
tags: [bro-gen, ios12, binding]
---
# iOS 12.2
iOS 12.2 has arrived as [PR363](https://github.com/MobiVM/robovm/pull/363).
It reflects changes [apple did](https://developer.apple.com/documentation/ios_release_notes/ios_12_2_release_notes?language=objc). Beside updating API robovm-cocoatouch received following updates:
- fixed broken structs in AudioToolBox;
- added IntentsUI framework;
- added UserNotificationsUI framework;

# What's new in bro-gen
<!-- more -->
Main change its support for inner and anonymous structures and unions. Previously bro-gen skipped these and it resulted in broken Structures at java side.  

## Case1: inner structs with member reference
```cpp
struct StructWithInner {
    int memberA;
    struct {
       int memberB;
    } innerA;
}
```

will be transformed into following:
```java
public class StructWithInner extends Struct<StructWithInner> {
    @StructMember(0) public native int getMemberA();
    @StructMember(0) public native StructWithInner setMemberA(int memberA);
    @StructMember(1) public native @ByVal StructWithInner$InnerStruct$1 getInnerA();
    @StructMember(1) public native StructWithInner setInnerA(@ByVal StructWithInner$InnerStruct$1 innerA);
}
```

`StructWithInner$InnerStruct$1` has to be configured in yaml to be generated, proper name can be assign there as well.

## Case2: anonymous struct
```cpp
struct StructWithInner {
    int memberA;
    struct {
       int memberB;
    };
}
```

Anonymous structure are flatten and their member become member of owning struct:
```java
public class StructWithInner extends Struct<StructWithInner> {
    @StructMember(0) public native int getMemberA();
    @StructMember(0) public native StructWithInner setMemberA(int memberA);
    @StructMember(1) public native int getMemberB();
    @StructMember(1) public native StructWithInner setMemberB(int memberB);
}
```

## Case3: anonymous unions
Anonymous unions are not being flatten and is extracted into standalone struct:
```cpp
struct StructWithInner {
    int memberA;
    union {
       int memberB;
    };
}
```

`StructWithInner$InnerStruct$1` has to be configured in yaml to be generated, proper name can be assign there as well. Also member name `autoMember$2` is generated for it, also might be configured in yaml.
```java
public class StructWithInner extends Struct<StructWithInner> {
    @StructMember(0) public native int getMemberA();
    @StructMember(0) public native StructWithInner setMemberA(int memberA);
    @StructMember(1) public native @ByVal StructWithInner$InnerStruct$1 getAutoMember$2();
    @StructMember(1) public native StructWithInner setAutoMember$2(@ByVal StructWithInner$InnerStruct$1 autoMember$2);
}
```
## Case4: when union is owner
When root structure is union -- all inner structures are being extracted. Anonymous structures are not being flatten.
```cpp
union UnionWithInner {
    int memberA;
    struct {
       int memberB;
    };
}
```
Result:  
```java
public class UnionWithInner extends Struct<UnionWithInner> {
    @StructMember(0) public native int getMemberA();
    @StructMember(0) public native UnionWithInner setMemberA(int memberA);
    @StructMember(0) public native @ByVal UnionWithInner$InnerStruct$1 getAutoMember$2();
    @StructMember(0) public native UnionWithInner setAutoMember$2(@ByVal UnionWithInner$InnerStruct$1 autoMember$2);
}
```

## Case5: inner structs that are not members
In example bellow `struct Inner` is not an anonymous member but structure definition. it is not a subject for member generation.
```cpp
struct StructWithInner {
    int memberA;
    struct Inner{
       int memberB;
    };
}
```
