---
layout: post
title: 'bugfix #230: Interface Builder: CGPoint/CGSize/CGRect/NSRange are not supported, wrong obj-c class name for empty @CustomClass()'
tags: [fix, 'interface builder']
---
**History** [IB integrator generates wrong class name for empty @CustomClass, doesn't support structs in IBInspectable](https://github.com/MobiVM/robovm/issues/230)  
**Fix** [PR231](https://github.com/MobiVM/robovm/pull/231)  

Have discovered that User define attributes/UIInspectable are not handled correctly: XCode project is not generated, if just added in code bypassing IB -- values are not set in case of field outlets. Structs that causes problem:
<!-- more -->
- CGPoint
- CGSize
- CGRect
- NSRange

***Part zero:*** Wrong class name  
Was under impression that empty value in `@CustomClass` will generated iOS class with simpleClassName() but that was wrong, runtime generates it as bellow:  
```java
//ObjCClass.java
private static String getCustomClassName(Class<? extends ObjCObject> type) {
    CustomClass customClassAnno = type.getAnnotation(CustomClass.class);
    String name = type.getName();
    if (customClassAnno != null && customClassAnno.value().length() > 0) {
        name = customClassAnno.value();
    } else {
        name = CUSTOM_CLASS_NAME_PREFIX + name;
    }
    name = name.replace('.', '_');
    return name;
}
```
To make IB generator compatible it was fixed. *But as for me it would be nice to generate simpleClassName() instead of fully qualified one, as it is more user friendly*

***Part one:*** Interface builder  
Fix of this part is straight forward, just added code to support these structs, check [PR231](https://github.com/MobiVM/robovm/pull/231)

***Part two:*** Compiler
Tricky part that outlets declared through setters work. Broken field one. Time ago outlet declaration on fields were added by me in [this pr](https://github.com/MobiVM/robovm/pull/109). When it was added there was outlet support through methods implemented. And to implement support for field I did a trick: synthesized java setter during SOOT path. And after that allowed existing code to transform this method into outlet property callback. The fix is to annotate parameter with `@ByVal` for struct setters during that run. Check changes in ObjCMemberPlugin.transformIBOutletFieldToSetters
