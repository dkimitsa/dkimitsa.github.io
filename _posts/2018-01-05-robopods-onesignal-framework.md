---
layout: post
title: 'RoboPods: OneSignal framework'
tags: ['bro-gen', binding, robopods, altpods]
---

New [`OneSignal.framework`](https://github.com/danielgindi/Charts) pod, currently it is in testing. Meanwhile it is available from my own `maven` repo for intermediate results `altpods`:
`Altpods` are available as repository in [github](https://github.com/dkimitsa/robovm-robopods), also it is deployed it to 'oss.sonatype.org' so it is available as maven artifact.

>OneSignal is a free push notification service for mobile apps

*Note:* it doesn't include `Notification Service Extension` as there is luck of support for this functionality in RoboVM itself. Will be added soon

<!-- more -->

### to use OneSignal pod configure your `robovm.xml`

```
<config>
    ...
    <frameworkPaths>
        <path>libs</path>
    </frameworkPaths>
    <frameworks>
        <framework>OneSignal</framework>
    </frameworks>
</config>
```

### Add the following dependency to your `build.gradle`:

```
dependencies {
   ... other dependencies ...
    compile "io.github.dkimitsa.robovm:robopods-onesignal-ios:$altpodsVersion"
}
```

Sample application is available [by link](https://github.com/dkimitsa/robovm-samples/tree/alt/robopods/onesignal/ios)  
