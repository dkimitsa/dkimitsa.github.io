---
layout: post
title: 'fix #561: wrong pointer to flexible array member'
tags: ['fix']
---
## Background
[issue #561 ExtAudioFile.read() isn't playing nice anymore](https://github.com/MobiVM/robovm/issues/561). `@yajirobe69` in comment described that possible root case it is a due field annotation was changed from `@Array({1})` to `@ByVal`.  
`buffers` field is a flexible array member of `AudioBuffer` struts. To work with this array required to get pointer to first struct (member itself) and then navigate in it by using methods of `Struct` class. 

## Root case 
Root case of issue was introduced in [e0b6db6](https://github.com/MobiVM/robovm/commit/e0b6db61adf82d3d0df95f93b14e70f88b4f187a). Accessing struct member is a struct kind and is annotated as `@ByVal` will cause following:  
* first element being accessed;
* copied;
* copy of it is returned

While it is normal to read struct by value, in flexible array member case it is wrong as:
* pointer to first struct is required, not pointer to the COPY;
* future operations with copy, e.g. setting elements (at index 1+) might cause memory corruption.

## The fix  
<!-- more -->  
`bro-gen` script was updated to put `@Array({1})` in case of flexible array (of struct) member.  
CocoaTouch bindings were re-generated.  
Changes proposed as [PR565](https://github.com/MobiVM/robovm/pull/565)

## Struct types and annotation mess
Consider simple `MIDIPacketList` struct with flexible array member as an example:  
```c
struct MIDIPacketList
{
	UInt32      numPackets;
	MIDIPacket  packet[1];
};
```

Items in `packet` are being accessed by getting pointer to first element and working with it:  
```c
  // getting pointer 
  MIDIPacketList *packetList = ..
  MIDIPacket *packets = packetList->packet; // or  &packetList->packet[0] 
  // accessing 
  MIDIPacket pkt; 
  packets[10] = pkt;      
```

Corresponding code for getting/setting items looks as bellow in RoboVM:  
```java
  MIDIPacketList packetList;
  MIDIPacket packets = packetList.getPacket();
  // get 
  packets.next(10);
  // set 
  packets.update(new MIDIPacket(), 10);
```

The question is how `getPacket()` is declared: 
### Case1: @ByRef
`@ByRef` means pass as pointer and is the default. Member is a POINTER to the struct:
 
```
@StructMember(1) public native @ByRef MIDIPacket getPacket();
// also equal by droping annotation 
@StructMember(1) public native MIDIPacket getPacket();
```

The binding in this case doesn't correspond initial struct and RoboVMs `packets = packetList.getPacket()` is equal to following `C` code: 
```c
// binding corresponds to the struct:
struct MIDIPacketList
{
	UInt32      numPackets;
	MIDIPacket  *packet;
};  

// packets = packetList.getPacket() is equal to
MIDIPacket *packets = packetList->packet;
```

As the binding uses broken presentation of original structure working with it as with flexible array member struct will produce memory corruption.

### Case2: @ByVal
`@ByVal` means pass by value.  
```
@StructMember(1) public native @ByVal MIDIPacket getPacket();
```

RoboVMs `packets = packetList.getPacket()` is equal to following `C` code:
```c
// packets = packetList.getPacket() is equal to
MIDIPacket packets = packetList->packet[0];
```

It returns by-value struct copy of first element, it doesn't represent an array. Getting pointer to it and working with it as with an array is wrong as point not to original struct member and will cause memory corruption/illegal access.    

### Case3: @Array({1})
> Bro provides the @Array annotation which is used to bind array struct members. 

Usage of `@Array({1})` seem to be valid and in place as we deal with a flexible array. The only messy thing here is type of array element:
* its array of Struct; 
* but Struct is not annotated with `@ByVal`;
* expected to be an array of pointers to struct;
* while it is not array of pointers (but array of structs) and it works. 

### CaseX: pointer
In additional to `@ByRef` there is special class in each struct `class MIDIPacketListPtr extends Ptr<MIDIPacketList, MIDIPacketListPtr>` that intended to represent a pointer to struct. 
The MESSY part here is that it is a STRUCT and might be accessible by `@ByRef` and `@ByVal` with different result. 
