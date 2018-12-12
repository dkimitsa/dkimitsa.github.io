---
layout: post
title: 'bro-gen: whats new after ios12 binding'
tags: ['bro-gen', whatsnew, binding]
---
bro-gen received several updates and new features during handling iOS12 bindings, mostly fixes but there are new ones. All changes are pushed to [github](https://github.com/dkimitsa/robovm-bro-gen).  
Whats new:  
<!-- more -->

## `typed_enums`
Apple started grouping constants into [typed enums](https://developer.apple.com/documentation/swift/objective-c_and_c_code_customization/grouping_related_objective-c_constants). Mostly in favor of Swift. This limit values that can be used with APIs (for example key for NSDictionary).
```
typedef NSString * CIImageOption NS_TYPED_ENUM;
...
CORE_IMAGE_EXPORT CIImageOption const kCIImageTextureTarget CI_GL_DEPRECATED_MAC(10_9,10_14);
CORE_IMAGE_EXPORT CIImageOption const kCIImageTextureFormat CI_GL_DEPRECATED_MAC(10_9,10_14);
..
+ (CIImage *)imageWithCGImage:(CGImageRef)image options:(nullable NSDictionary<CIImageOption, id> *)options;
```
In RoboVM cocoa binding there are `NSDictionaryWrapper` and `GlobalValueEnumeration` to subclass that were used long time for same purpose. But to make bro to generate class for it was required to create Values configurations:
```
values:
    kCIImageTexture(Target|Formate):
        dictionary: CIImageOptions
        name: '${g[0]}'
        type: NSString
```

New functionality allows to define typed enum/dictionary based on typedef and not set of keys:
```
typed_enums:
    CIImageOption:
        dictionary: CIImageOptions
        type: NSString
```

Definition is similar to one used for values grouping but also adds 'prefix' and 'suffix' parameters that allows removing extra texts from items;

## `generic_typedefs`
Configuration key intended to override type resolution used in generics. On of the reason for it that some types are resolved (through typedefs) to deeply to java one that these are not compatible with cocoa anymore, for example NSString being converted to Java String: `NSDictionary<String, ?>` which makes it fail to compile.
Before this change it was required to manually setup return/parameter types for methods. Today `foundation.yaml` contains following code:
```
typedefs:
    'NSString *': String
generic_typedefs:
    'NSString *': NSString
```

## inherited initializers exposed
Bro-ben was creating constructors for classes only if these were present in corresponding objc class. For example `initWithFrame` is missing in `UIButton` but declared in `UIView` as `-(instancetype)initWithFrame:(CGRect)frame;` and is available for `UIButton`. This change exposed lot of hidden constructors.

## inherited configuration
This feature automatically searches for method configuration is super class. This reduce amount of configuration to be written in case such method can be found in one of super classes. Bright example is `-initWithCoder:` which is declared in many files but configuration for it can be picked from root protocol.

## better suggestion
Suggestion updated to:
- not notify about missing configuration for method in class if this method is from protocol (it is enough to setup it for protocol);
- it will suggest to proper name any methods that has `objc` specific `With` text. E.g. `encodeWithData` shall be named java-style `encode`;
- as result of inherited configuration feature it will mark methods with missing configuration that are also present in super classes. As it is enough now to configure it in lowest class in hierarchy.


## Deprecation/availability messages now are written to `javadoc` section:
```
/**
 * @since Available in iOS 6.0 and later.
 * @deprecated Deprecated in iOS 12.0. Use [SKDownload state] instead
 */
```

## `skip_implements`
Very specific parameter for protocols. When protocol is marked with `skip_implements: true` all objects that implement it will not have reference to this protocol in `implements` section but protocol methods will be included into class. This is very specific feature was required for `CKRecordKeyValueSetting` protocol and `CKRecord` only/

## Other fixes
- Constant status resolution sometimes not comes through. This was leading to lot of setter for read-only values;
- fix for crash due unhandled cursor type `417=CXCursor_VisibilityAttr`, just ignoring
- fixed enum item naming when there was only one item in enum. It was not able to find out common prefix. Now enum name is used as common prefix in this case.
