---
layout: post
title: 'Framework target: EXC_BAD_ACCESS on NPE in try-catch block'
tags: ['target-framework', fix]
---
Application with RoboVM target crashes with EXC_BAD_ACCESS, investigation shows that this happens on null pointer exception (NPE) that enclosed in corresponding try-catch block and shall not break away? Already covered in [Tutorial: Crash Reporters and java exceptions]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %}).   
Root case is same -- `signals`.  
Solution is same -- use `Signals.installSignals`.  
Post bellow describes how to deal with it and introduces changes into framework template.  
<!-- more -->
## API for host application to `Signals.installSignals`
To obey RoboVM's rules host (objc or swift) application shall has access to `Signals.installSignals`. Adding it to 'SampleSDK' interface of sample application:
```java
public final class Api {
    /**
     * this protocol is main class protocol that root entry point to SDK
     */
    interface SampleSDK extends ObjCProtocol {
        ...

        /**
         * installs Signal handlers of main application
         * Host project shall install all signal handlers inside in installer callback implementation
         * This will allow RoboVM keeps own signal handlers and handle NPE
         * @param installer block were all crash analytics initialization shall happen
         */
        @Method void installSignals(@Block Runnable installer);

        ...
    }
}
```

Its implementation is straight forward:
```java
public class SampleSDKImpl extends NSObject implements Api.SampleSDK {
    ...

    @Override
    public void installSignals(@Block Runnable installer) {
        Signals.installSignals(installer::run);
    }
}
```

## Changes to host application
Host application shall install all crash reporters inside installer block:
```objc
- (BOOL)application:(UIApplication * )application didFinishLaunchingWithOptions:(NSDictionary * )launchOptions {
    // Override point for customization after application launch.

    // testing RoboVM framework
    SampleSDK* sdk = SampleSDKInstance();
    [sdk installSignalHandlers:^{
        [[Fabric sharedSDK] setDebug:true];
        [[Crashlytics sharedInstance] setDebugMode:true];
        [Fabric with:@[[Crashlytics class]]];
    }];

    return YES;
}
```

RoboVM's framework template was updated with same changes with [commit](https://github.com/MobiVM/robovm/pull/300/commits/9f4b6aa6bfe9c70578f49c7862a35d8d1e001d16), samples updated with [another commit](https://github.com/dkimitsa/robovm-samples/commit/c369c234efed1d188c0f4c1171a18a90500e332b).   
Related:
* [Tutorial: Crash Reporters and java exceptions]({{ site.baseurl }}{% post_url 2018-02-16-crash-reporters-and-java-exceptions %})
* [Tutorial: writing Framework using improved framework target]({{ site.baseurl }}{% post_url 2018-01-16-tutorial-writing-framework-improved %})
