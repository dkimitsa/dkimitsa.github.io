---
layout: post
title: 'aggressive treeshaker: fixing ObjectOutputStream.writeObject()'
tags: ['tree shaker', enhancement]
---

Aggressive tree shaker helps to reduce application footprint by removing all methods that are not referenced in code. This often breaks reflection-based code and required classes has to be explicitly specified using `<forceLinkClasses>` but sometimes it is not enough. While working with [nitrite-database](https://github.com/dizitart/nitrite-database) I faced case when class is referenced but some methods of it is still removed. Here is an example:
```java
private void testSerialize() {
        try {
            new ObjectOutputStream(new ByteArrayOutputStream()).writeObject(new HashMap<>());
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```
fails with:
```
java.io.InvalidClassException: java.util.HashMap doesn't have a field loadFactor of type float
	at java.io.ObjectOutputStream.writeFieldValues(ObjectOutputStream.java:955)
```

This happens due `internal` private and unreferenced methods got removed from `HashMap`.

## java.io.Serializable contract  
As specified in [documentation](https://docs.oracle.com/javase/7/docs/api/java/io/Serializable.html) there is special case:
<!-- more -->

> Classes that require special handling during the serialization and deserialization process must implement special methods with these exact signatures:
```
 private void writeObject(java.io.ObjectOutputStream out)
     throws IOException
 private void readObject(java.io.ObjectInputStream in)
     throws IOException, ClassNotFoundException;
 private void readObjectNoData()
     throws ObjectStreamException;
```

`HashMap` is designed with for this special handling but `ObjectOutputStream` can't find these methods runtime as these were aggressively removed by tree shaker and thread this object as regular one and tries to serialize it using reflection. But there is an conflict in field definition which causes exception. Anyone serialization is broken.

## the fix -- `forceLinkMethods`
Straightforward solution is to tell tree shaker to keep these methods. I've added following [PR322](https://github.com/MobiVM/robovm/pull/322) that adds this functionality and allows flexible way specifying methods to be force linked. It has to be configured using `robovm.xml` using `<forceLinkMethods>` tag.  

Under `<forceLinkMethods>` one ore more `<entry>` tags follow specifying class and method pairs:  
```xml
<forceLinkMethods>
    <entry>
        <!-- ... -->
    </entry>
    <entry>
        <!-- ... -->
    </entry>
</forceLinkMethods>
```

Entry contains following tags:

* Class list where to look for methods:  

```xml
<owners>
    <pattern extendable="true">java.io.Serializable</pattern>>
</owners>
```

Class name pattern is specified by `<pattern>` tag and its Ant pattern. If `extendable=true` attribute is specified then class will be checked against super classes and interfaces it (and its super) implements. Otherwise only classes that match pattern are subject for method force-linking.

* Methods that has to be force-linked in matching classes:  

```xml
  <methods>
    <method>writeObject(Ljava/io/ObjectOutputStream;)V</method>
    <method>readObject(Ljava/io/ObjectInputStream;)V</method>
 <methods>
```

Each method item is concatenation of method name with its full JVM signature, e.g.:
> writeObject(Ljava/io/ObjectOutputStream;)V

corresponds to:
> void writeObject(ObjectOutputStream stream)

## packing everything together
Following configuration entry reverts makes ObjectOutputStream.writeObject() work again:  
```xml
     <!--Force link methods required for ObjectOutputStream to operate-->
    <forceLinkMethods>
        <entry>
            <methods>
                <method>writeObject(Ljava/io/ObjectOutputStream;)V</method>
                <method>readObject(Ljava/io/ObjectInputStream;)V</method>
            </methods>
            <owners>
                <pattern extendable="true">java.io.Serializable</pattern>>
            </owners>
        </entry>
    </forceLinkMethods>
```
