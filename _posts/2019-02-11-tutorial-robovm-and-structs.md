---
layout: post
title: 'tutorial: RoboVM and Struct pointers mess'
tags: ['bro-gen', tutorial, binding]
---
A struct in the C is a composite data type placed under one name in a block of memory, allowing the different variables to be accessed via a single pointer [wiki](https://en.wikipedia.org/wiki/Struct_(C_programming_language)).   
The Bro Java to Native Bridge [supports](https://github.com/MobiVM/robovm/wiki/The-Bro-Java-to-Native-Bridge#structs) structs data type. And there are several scenarios how they are used:  
<!-- more -->

## Functiona parameters: @ByRef and @ByVal
In C world there are two ways to pass struct to function:  
By value -- struct is passed as parameter (it is copied and the copy is passed): 
```c
// c-code
typedef struct {float f;} StructF;
void byValue(StructF s) {
    s.f = 5.4f;
}

void main() {
    StructF ss = {1.0};
    byValue(ss);  // ss is not changed
}
```

By reference(pointer) -- pointer to struct is passed as parameter and no copy being made. All changes to struct made in function affects original struct being passed.
```c
// c-code
void byPointer(StructF *s) {
    s.f = 5.4f;
}

void main() {
    StructF ss = {1.0};
    byValue(&ss);  // ss.f == 5.4 now 
}
```

In RoboVM Java world there is no such thing as Struct and it is represented by corresponding Class and class instances are passed as references. To tell Bro bridge what code to generate `@ByRef` and `@ByVal` annotations are used: 
```java
// java-code
native void byValue(@ByVal StructF s); // will make a copy and pass by value
native void byPointer(@ByRef StructF s); // will pass a pointer
native void byPointer2(StructF s); // default: also will pass a pointer
```

## Function return type: @ByRef and @ByVal
Similary native function can return struct (`@ByVal`) or pointer to it (`@ByRef`). Specifying valid is requred to get valid result:
```java
// java-code
native @ByVal StructF returnByValue(); 
native @ByRef StructF returnByPointer(); 
native StructF alsoReturnByPointer(); 
```

## Array of Structs
Array of structures is C is passed and returned by pointer to first struct:
```c
// c-code
typedef struct {float f;} StructF;
void parameterStructArray(StructF *s, size_t size) {
    s[size - 1].f = 0;
}

StructF someArray[1000];
StructF* returnStructArray() {
    return someArray;
}

void main() {
    StructF ss[1000] = {0};
    parameterStructArray(ss, 1000); 

    StructF ret = returnStructArray();
    ret[0].f = 0;
}
```

As in C world these are pointers -- in RoboVM Java world these have to be used as `@ByRef`:
```java
// java-code
native void parameterStructArray(@ByRef StructF s, long size); 
native @ByRef StructF returnStructArray(); 
```

There is no RoboVM class to reflect array of structures. This functionality is part of base `org.robovm.rt.bro.Struct`. Creating array of required size:
```java
// java-code
// creating VectorFloat2[100]
int size = 100;
VectorFloat2 structArray = VectorFloat2.allocate(VectorFloat2.class, 100)
// to set data structArray.update() APIs 
```

## iterrating over array of Structs in RoboVM
To iterrate over array of struct that represented by subclass of `org.robovm.rt.bro.Struct` it is required to know size of array. There are multiple options for this:
```java
// java-code
int size = 100;
VectorFloat2 structArray = VectorFloat2.allocate(VectorFloat2.class, size)
// convert to java array
VectorFloat2 asArray[] = structArray.toArray(size);
// convert to java list 
List<VectorFloat2> asList[] = structArray.toList(size);
// get the iterrator 
Iterator<VectorFloat2> it = structArray.iterator(size);
```

## BEWARE: Struct.Ptr
Bro-gen generates static inner classes for each struct with Ptr suffix, such as:  
```java
public static class VectorFloat2Ptr extends Ptr<VectorFloat2, VectorFloat2Ptr> {}
```

There few moments about it:
- it shall be used with `@ByVal` annotation most of the time. As `org.robovm.rt.bro.ptr.Ptr` is a Struct itself. And if it used as `@ByRef` it represent pointer to pointer to struct.   
- it is not sutable for iterating over arrays of structs, as Ptr is standalone struct and all iteration API will iterrate on array of pointers to structs.  

```
// java-code
native @ByVal VectorFloat2.VectorFloat2Ptr getPointer();
void foo() {
    VectorFloat2.VectorFloat2Ptr ptr = getPointer();
    // get first element
    VectorFloat2 value = ptr.get();
    // if pointer to array -- get iterator from first element
    VectorFloat2 arr[] = value.toArray(100);
}
```

## Fixing array issue in RoboVm CocoaTouch
Due to the bug in `bro-gen` there was lot of wrong bindings generated for Pointers to vectorized structs. [PR348](https://github.com/MobiVM/robovm/pull/348) fixes these issues.

