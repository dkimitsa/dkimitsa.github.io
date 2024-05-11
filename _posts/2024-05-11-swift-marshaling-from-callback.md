---
layout: post
title: 'Swift: when ld and runtime class name differs'
tags: [investigation, runtime, swift]
---
Issue observed when swift call back to java code and provided object from swift side failed to marshal to destination class: 
> java.lang.ClassCastException: org.robovm.apple.foundation.NSObject cannot be cast to org.bindings.SomeSwiftClass

This happens when corresponding class is not loaded at Java side and RoboVM's ObjCRuntime doesn't recognize its pointer.

Lets investigate.
<!-- more -->

## Background

Class is declared on Swift side as:
```swift
@objc public class SomeSwiftClass: NSObject {
...
}
```

Framework exports it as:
```
00000000000389a8 S _OBJC_CLASS_$__TtC9NameSpace14SomeSwiftClass
000000000003a698 D _OBJC_METACLASS_$__TtC9NameSpace14SomeSwiftClass
```

And in bindings it looks as bellow:
```java
@NativeClass("_TtC9NameSpace14SomeSwiftClass")
public class SomeSwiftClass extends NSObject {
  static {
    ObjCRuntime.bind(SomeSwiftClass.class);
  }
  ...
}
```

## the issue 
While this class can be created from Java side without the issue:
> new SomeSwiftClass()

it fails when it comes about receiving it in callback:
```
SwiftSdk.shared().doWithCallback((new VoidBlock1<SomeSwiftClass>() {
      @Override
      public void invoke(SomeSwiftClass payload) {
          ...
      }
)

> java.lang.ClassCastException: org.robovm.apple.foundation.NSObject cannot be cast to org.bindings.SomeSwiftClass 
```

this happens when `SomeSwiftClass` is not yet known to ObjCRuntime (if we do `new SomeSwiftClass()` initially it will work).

## why `new SomeSwiftClass()` works
Long story short -- it calls `objc_getClass("_TtC9NameSpace14SomeSwiftClass")` and it works.

When new instance of `SomeSwiftClass` is created, runtime has to call objc equal code `[[SomeSwiftClass alloc] init]` and its done his way when class is not loaded:
```java
protected long alloc() {
    long h = this.getHandle();
    if (h == 0L) {
        h = alloc(this.getObjCClass());
    }
    return h;
}
public final ObjCClass getObjCClass() {
  return ObjCClass.getFromObject(this);
}
public static ObjCClass getFromObject(ObjCObject id) {
  //....
  return getByType(id.getClass());
}
public static ObjCClass getByType(Class<? extends ObjCObject> type) {
  //....
  NativeClass nativeClassAnno = type.getAnnotation(NativeClass.class);
  if (nativeClassAnno != null) {
    name = nativeClassAnno.value();
    name = "".equals(name) ? type.getSimpleName() : name;
    long classPtr = ObjCRuntime.objc_getClass(VM.getStringUTFChars(name));
    if (classPtr != 0L) {
      c = new ObjCClass(classPtr, type, name, false, false);
    }
    //...
  }
  return c;
}
```

It successfully able to get pointer to class using `objc_getClass("_TtC9NameSpace14SomeSwiftClass")`.

## why callback doesn't work
Long story short:
- expected objc name `_TtC9NameSpace14SomeSwiftClass`
- but runtime name is `NameSpace.SomeSwiftClass`
- because @objc attribute in swift code was used without argument

When native pointer comes from native side, RoboVM ObjCRuntime has to find Java class that corresponds to ObjC class and create instance of it.
For not-loaded before class it goes this way:
```java
public static class Marshaler {
  @MarshalsPointer
  public static ObjCObject toObject(Class<? extends ObjCObject> cls, long handle, long flags) {
    ObjCObject o = ObjCObject.toObjCObject(cls, handle, 0);
    return o;
  }
}
public static <T extends ObjCObject> T toObjCObject(Class<T> cls, long handle, int afterMarshaledFlags, boolean forceType) {
  //...
  ObjCClass objCClass = ObjCClass.getFromObject(handle, expectedType != cls);
  //...    
  return createInstance(objCClass, handle, afterMarshaledFlags, true);
}

public static ObjCClass getFromObject(long handle, boolean optional) {
  long classPtr = ObjCRuntime.object_getClass(handle);
  return toObjCClass(classPtr, optional);
}
public static ObjCClass toObjCClass(final long handle, final boolean optional) {
  long classPtr = handle;
  ObjCClass c = ObjCObject.getPeerObject(classPtr);
  if (c == null) {
    c = getByNameNotLoaded(VM.newStringUTF(ObjCRuntime.class_getName(classPtr)));
  }
  //...
  while (c == null && classPtr != 0L) {
    classPtr = ObjCRuntime.class_getSuperclass(classPtr);
    // ... do getByNameNotLoaded(...) for all supers 
  }
  //....
  return c;
}
private static ObjCClass getByNameNotLoaded(String objcClassName) {
  Class<? extends ObjCObject> cls = allNativeClasses.get(objcClassName);
  if (cls != null) {
    return getByType(cls);
  }
  // ...
  return null;
}
```

And result is, that `allNativeClasses[class_getName(classPtr)] == null` for native object class. In this case it goes for all supers and picks `NSObject` as best known class.  
But `allNativeClasses` is being populated in static block of `ObjCClass` -- it retrieves all `ObjCObject` subclasses with `VM.listClasses(ObjCObject.class)` and checks for `@NativeClass("_TtC9NameSpace14SomeSwiftClass")` annotations and fills `allNativeClasses` map with it, and it indeed contains
>allNativeClasses["_TtC9NameSpace14SomeSwiftClass"] = SomeSwiftClass.class

Issue here is that `class_getName(classPtr)` returns runtime name of class as `NameSpace.SomeSwiftClass` and it is not expected one `_TtC9NameSpace14SomeSwiftClass`.
Its dues how Swift code is written using [@objc attribute](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/attributes/#objc):
> Note
> The argument to the objc attribute can also change the runtime name for that declaration.  
> ..  
> If you specify a name by passing an argument, that name is used as the name in Objective-C code and as the runtime name. 
> If you omit the argument, the name used in Objective-C code matches the name in Swift code, and the runtime name follows the normal Swift compiler convention of name mangling.

NB: probably there is mistake, as if argument is omitted Objective-C code matches mangled name and runtime matches Swift code, asked [at swift forum](https://forums.swift.org/t/objc-omitted-argument-note-might-be-not-correct/71768). 

Back to swift code:
> @objc public class SomeSwiftClass: NSObject {

There is no argument -- as result ObjC side work as expected and its runtime name is not known for RoboVM. 

## `NameSpace.SomeSwiftClass` works with obj-c runtime API
Long story short: obj-c runtime looks up for a name as it is, if fails then consider name as swift one, mangles and tries again.

Unexpected, but `objc_getClass("NameSpace.SomeSwiftClass")` loads and return ObjC class while it is declared as `_TtC9NameSpace14SomeSwiftClass` in Objective-C code. 
Opensource Apple code for [objc4](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-runtime.mm.auto.html) explains why its possible:
```cpp
Class objc_getClass(const char *aClassName)
{
    return look_up_class(aClassName, NO, YES);
}
```
[look_up_class](https://opensource.apple.com/source/objc4/objc4-723/runtime/objc-runtime-new.mm.auto.html):
```cpp
Class 
look_up_class(const char *name, 
              bool includeUnconnected __attribute__((unused)), 
              bool includeClassHandler __attribute__((unused)))
{
    //...
    result = getClass(name);
    return result;
}
static Class getClass(const char *name)
{
    runtimeLock.assertLocked();

    // Try name as-is
    Class result = getClass_impl(name);
    if (result) return result;

    // Try Swift-mangled equivalent of the given name.
    if (char *swName = copySwiftV1MangledName(name)) {
        result = getClass_impl(swName);
        free(swName);
        return result;
    }

    return nil;
}
```

It works as in case of failure it gives a second change, assuming name is swift one, mangles it before second lookup. 
These changes were added 2014 with iOS8 and Swift v1 release.

## what about Swift itself? 

Long story short:
- in opposite to iOS8 ObjC4 implementation it uses `objc_setHook_getClass`, demangle name if required. Looks by Swift name.

Swift can also take part of `objc_getClass` process. It actively used `objc_setHook_getClass` API added in iOS12.2. That API allows to intercept `objc_getClass` and do own class lookup.
ObjectiveC runtime lookup code was [changed](https://opensource.apple.com/source/objc4/objc4-818.2/runtime/objc-runtime-new.mm.auto.html):
```cpp
Class 
look_up_class(const char *name, 
              bool includeUnconnected __attribute__((unused)), 
              bool includeClassHandler __attribute__((unused)))
{
    // ... returns already realized class if any
    if (!result) {
        // Ask Swift about its un-instantiated classes.
        // ...
        // Call the hook.
        Class swiftcls = nil;
        if (GetClassHook.get()(name, &amp;swiftcls)) {
            ASSERT(swiftcls-&gt;isRealized());
            result = swiftcls;
        }
        // ...
    }

    return result;
}
```

Swift does a bit of things in hook as it can be seen from [source](https://github.com/apple/swift/blob/84983353d3ac3a5c3ada623bd811ff875609b0a1/stdlib/public/runtime/MetadataLookup.cpp#L3040):
```cpp
__attribute__((constructor))
static void installGetClassHook() {
  if (SWIFT_RUNTIME_WEAK_CHECK(objc_setHook_getClass)) {
    SWIFT_RUNTIME_WEAK_USE(objc_setHook_getClass(getObjCClassByMangledName, &OldGetClassHook));
  }
}


static BOOL
getObjCClassByMangledName(const char * _Nonnull typeName,
                          Class _Nullable * _Nonnull outClass) {
  // Demangle old-style class and protocol names, which are still used in the
  // ObjC metadata.
  StringRef typeStr(typeName);
  const Metadata *metadata = nullptr;
  if (typeStr.starts_with("_Tt")) {
    Demangler demangler;
    auto node = demangler.demangleSymbol(typeName);
    // ...
    metadata = swift_getTypeByMangledNode(
        // ...
    ).getType().getMetadata();
  } else {
    if (validateObjCMangledName(typeName))
      metadata = swift_stdlib_getTypeByMangledNameUntrusted(typeStr.data(),
                                                            typeStr.size());
  }
  if (metadata) {
    // ...
    if (objcClass) {
      *outClass = objcClass;
      return YES;
    }
  }

  return OldGetClassHook(typeName, outClass);
}
```

Here we can see opposite to obj4 process:
- it detects mangled names such as `_TtC9NameSpace14SomeSwiftClass`
- converts it to Swift code name like `NameSpace.SomeSwiftClass`
- loads using own metadata.

# Bottom line

Regarding `objc_getClass`:
- iOS8..12.2 -- ObjectiveC runtime will mangle Swift name and try again if class not resolved on first try;
- iOS12.2+ -- Swift will try to demangle name if mangled provided. And does own lookup;

Regarding problem:
- Problem doesn't exist if Swift code defines ObjectiveC name in `@objc` argument;
- RoboVM do have a problem when runtime and objc class names are different and bindings uses ObjectiveC name;
- Solution would be to use Swift code names (e.x. `NameSpace.SomeSwiftClass`) with `@NativeClass` annotations.

# Fix / What to do

## If you owner of Swift code 
Use `@objc` attribute with argument to provide single ObjectiveC and runtime class names:
> @objc(SomeSwiftClass) public class SomeSwiftClass: NSObject {


## If you binding owner: 
Demangle any mangled Swift name for classes and to use it in `@NativeClass` annotations, e.g.:
>@NativeClass("NameSpace.SomeSwiftClass")
>public class SomeSwiftClass extends NSObject {

## If you use bindings but can't change them
There is a dirty reflection hack to register these classes in your code:
```java
try {
    // FIXME: hack to register run-time Swift name of classes to allow ObjCRuntime to
    //        marshal them to Java
    Field f = ObjCClass.class.getDeclaredField("allNativeClasses");
    f.setAccessible(true);
    Map<String, Class<? extends ObjCObject>> allNativeClasses = (Map<String, Class<? extends ObjCObject>>) f.get(null);
    allNativeClasses.put("NameSpace.SomeSwiftClass", SomeSwiftClass.class);
    allNativeClasses.put("NameSpace.AnotherSomeSwiftClass", AnotherSomeSwiftClass.class);
} catch (NoSuchFieldException | IllegalAccessException e) {
    throw new RuntimeException(e);
}
```

## if number of binding is too huge:
There is a way to patch RoboVM code to let it works -- it will register class to a demangled name as well:
```patch
Index: compiler/objc/src/main/java/org/robovm/objc/ObjCClass.java
IDEA additional info:
Subsystem: com.intellij.openapi.diff.impl.patch.CharsetEP
<+>UTF-8
===================================================================
diff --git a/compiler/objc/src/main/java/org/robovm/objc/ObjCClass.java b/compiler/objc/src/main/java/org/robovm/objc/ObjCClass.java
--- a/compiler/objc/src/main/java/org/robovm/objc/ObjCClass.java	(revision aaf2714ced264d8ce3cae9f2e0699359a0f2c602)
+++ b/compiler/objc/src/main/java/org/robovm/objc/ObjCClass.java	(date 1715368317743)
@@ -91,6 +91,25 @@
                 String name = nativeClassAnno.value();
                 if (name.length() == 0) {
                     name = cls.getSimpleName();
+                } else {
+                    // special case for ObjC class from Swift world that is declared without ObjC name.
+                    // e.g.         @objc public class SomeClass: NSObject
+                    // instead of   @objc(SomeClass) public class SomeClass: NSObject
+                    //
+                    // in this case it receives mangled namespace + class name as exported symbol:
+                    //   `_TtC8SomeUnit9SomeClass`  (and this value is present in @NativeClass annotation)
+                    // but it's class name (returned by class_getName()) will be set to swift one:
+                    //   `SomeUnit.SomeClass`
+                    // RoboVM can load this class by mangled name using objc_getClass("_TtC8SomeUnit9SomeClass").
+                    // but RoboVM can't recognize this class when it comes from native side, e.g. as callback parameter.
+                    // Because className (returned by class_getName()) to find corresponding native class. And its
+                    // value `SomeUnit.SomeClass` is not known as this class is registered with `_TtC8SomeUnit9SomeClass`
+                    // name
+                    // Solution: demangle name into proper Swift one and register class in allNativeClasses
+                    //           under both mangled and swift names
+                    String swiftName = demangleSwiftName(name);
+                    if (swiftName != null)
+                        allNativeClasses.put(swiftName, cls);
                 }
                 allNativeClasses.put(name, cls);
             } else {
@@ -128,6 +147,48 @@
     static boolean isObjCProxy(Class<?> cls) {
         return (cls.getModifiers() & ACC_SYNTHETIC) > 0 && cls.getName().endsWith(OBJC_PROXY_CLASS_SUFFIX);
     }
+
+    /**
+     * demangle swift class name
+     * sample: _TtC9Namespace9Classname => Namespace.Classname
+     * @param name - mangled name
+     * @return swift name or null if failed
+     */
+    public static String demangleSwiftName(String name) {
+        int len = name.length();
+        if (len <= 4 || !name.startsWith("_TtC"))
+            return null;
+        int idx = 4;
+        int chunkLen = 0;
+        StringBuilder sb = null;
+        do {
+            char c = name.charAt(idx);
+            if (c >= '0' && c <= '9') {
+                // still in length
+                chunkLen *= 10;
+                chunkLen += c - '0';
+                idx += 1;
+            } else {
+                // chunk data
+                if (chunkLen == 0 || idx + chunkLen > len)
+                    return null;
+                // create builder on first chunk, otherwise put separator
+                if (sb == null) sb = new StringBuilder();
+                else sb.append('.');
+                sb.append(name, idx, idx + chunkLen);
+                idx += chunkLen;
+                if (idx == len) {
+                    // last item and matches string size
+                    return sb.toString();
+                }
+                chunkLen = 0;
+            }
+
+        } while (idx < len);
+
+        // should not get here
+        return null;
+    }
     
     public static class Marshaler {
         @MarshalsPointer

```

# Happy coding  
