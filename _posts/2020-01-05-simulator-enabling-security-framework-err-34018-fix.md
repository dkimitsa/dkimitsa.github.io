---
layout: post
title: "Simulator: enabling Security framework(fix for error -34018)"
tags: ["fix"]
---

@Tom-ski reported that Security api like `SecItemAdd` fails in RoboVM applications while running on simulator with error -34018:
> The operation couldn’t be completed. (OSStatus error -34018 - Client has neither application-identifier nor keychain-access-groups entitlements)

Same time there is no issue for application when running on device. There are multiple reports about similar issues on web, and one on [Stackoverflow](https://stackoverflow.com/questions/20344255/secitemadd-and-secitemcopymatching-returns-error-code-34018-errsecmissingentit) contain reference to statement from [Xcode8.1 release notes](https://developer.apple.com/library/archive/releasenotes/DeveloperTools/RN-Xcode/Chapters/Introduction.html#//apple_ref/doc/uid/TP40001051-CH1-SW24):
> Keychain APIs may fail to work in the Simulator if your entitlements file doesn’t contain a value for the application-identifier entitlement.

Quick test reproduced this issue with Java code like this:
```java
SecAttributes attributes = new SecAttributes();
attributes.set(SecQuery.Keys.Class(), SecClass.Values.GenericPassword());
attributes.setAccount("My key");
attributes.set(SecValue.Keys.Data(), new CFString("My secretive bee 🐝"));
attributes.setAccessible(SecAttrAccessible.WhenUnlocked);
try {
    CFType res = SecItem.add(attributes);
    System.out.println(res);
} catch (OSStatusException e) {
    e.printStackTrace();
}
```

But there is an error possible in bindings, to check this native library with simple class was generated and called from RoboVM project:  
```objc 
@implementation SimpleTest
-(void) test {
    NSMutableDictionary *queryAdd = [NSMutableDictionary dictionary];
    [queryAdd setValue:kSecClassGenericPassword forKey:kSecClass];
    [queryAdd setValue:itemKey forKey:kSecAttrAccount];
    [queryAdd setValue:[itemValue dataUsingEncoding:NSASCIIStringEncoding] forKey:kSecValueData];
    [queryAdd setValue:kSecAttrAccessibleWhenUnlocked forKey:kSecAttrAccessible];
    int resultCode = SecItemAdd(queryAdd, nil);
    NSLog(@"SecItemAdd %d", resultCode);
}
@end
```

And result was same error -34018.

## the fix
Same time this ObjC code successfully runs on simulator from Xcode. That means there is something in build/signing process missing in case of RoboVM.
Looking throw Report navigator shows following:
- Xcode signs simulator binary with ADHOC identity. But it is not enough;
- Xcode adds entitlements during signing. But signing with simple `get-task-allow` (as mentioned in web) crashes App on start with `Invalid entitlements` exception.
- It turns out that Xcode uses `com.apple.security.get-task-allow` instead. With it app doesn't crash but Security API still doesn't work;
- Inspecting binary built by Xcode for `application-identifier` discloses another entitlements (besides one in signature) stored in `__TEXT  __entitlements` and it contains following:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>application-identifier</key>
	<string>6THE3YW9HQ.bundle.id</string>
	<key>keychain-access-groups</key>
	<array>
		<string>6THE3YW9HQ.bundle.id</string>
	</array>
</dict>
</plist>
```

Having such section embedded into RoboVM application solves the issue. Success !

## Bottom line
As result:
- signing of binary built for simulator is not required, but was added to be compatible with what Xcode is doing;
- RoboVM compiler now embeds __entitlement section similar to Xcode (but this is not documented anywhere).

Code was delivered as [PR447](https://github.com/MobiVM/robovm/pull/447)


