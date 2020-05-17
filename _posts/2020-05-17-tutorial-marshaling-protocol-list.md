---
layout: post
title: 'tutorial: marshaling NSArray of protocols'
tags: ['bro-gen', tutorial, binding]
---
Backtrack: [issue](https://github.com/dkimitsa/robovm/pull/1), [gitter](https://gitter.im/MobiVM/robovm?at=5ec0463a347a503172f5710d)  

It becomes problematic to marshal methods that return `NSArray<id<PROTOCOL>>` like bellow:  
```objc
- (void)readerSession:(NFCReaderSession *)session 
        didDetectTags:(NSArray<__kindof id<NFCTag>> *)tags;
```

### Issue 1: NSArray<T extends NSObject>  
Direct binding will not compile:
<!-- more -->
```java
@Method(selector = "tagReaderSession:didDetectTags:")
void didDetectTags(NFCTagReaderSession session, NSArray<?> tags);
```

Template argument type of `NSArray` should extend `NSObject` and protocol actually doesn't. Quick fix (wrong) -- marshall to list by with the custom marshaller:
```java
@Method(selector = "tagReaderSession:didDetectTags:")
 void didDetectTags(NFCTagReaderSession session, @org.robovm.rt.bro.annotation.Marshaler(NSArray.AsListMarshaler.class) List<NFCTag> tags);
```  

### Issue 2: ClassCastException NSObject cannot be cast to NFCTag  
NSArray was marshalled into the List, but it knows nothing about data it wraps. It marshalls as generic ObjC object. And here is simplified flow:
- it gets object class and looks for its java counter part;
- if not founds it goes to super one and repeats. 

There are three outcomes possible: 
- counter part exists then it will be marshalled to it;
- it downs to NSObject supper (skipping all not known classes) and marshall to it;
- class not inherited from NSObject -- then marshall will fail with exception (rare).

In the list above there is nothing about protocols due following:
- protocol -- is not a class;
- class might implementing multiple protocols;

As result direct marshaller of objc pointer into single Protocol proxy not possible.


### A possible fix:
For any missing class implementing ObjC protocols RoboVM objc runtime should create `java.lang.reflect.Proxy` with all protocols (interfaces) implemented. This will solve ClassCastException at java side. However each native method invocation will require `InvocationHandler` actions.  

### Today fix:  
Implement marshaller of particular list of protocol and marshall it manually. There is `ObjCObject.Marshaler.protocolToObject` that marshall pointer to proxy implementation of particular protocol. Just need wrap up things and marshall all items of NSArray: 
```java
class AsListMarshaller {
    @SuppressWarnings("unchecked")
    @MarshalsPointer
    public static List<NFCTag> toObject(Class<? extends NSObject> cls, long handle, long flags) {
        NSArray<NSObject> o = (NSArray<NSObject>) NSObject.Marshaler.toObject(cls, handle, flags);
        if (o == null) {
            return null;
        }
        List<NFCTag> list = new ArrayList<>();
        for (NSObject t : o) {
            NFCTag tag = (NFCTag) ObjCObject.Marshaler.protocolToObject(NFCTag.class, t.getHandle(), 0);
            list.add(tag);
        }
        return list;
    }
}
```
