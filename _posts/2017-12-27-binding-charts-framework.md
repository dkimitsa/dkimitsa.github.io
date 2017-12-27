---
layout: post
title: 'RoboPods: Charts framework'
tags: ['bro-gen', binding, robopods, altpods]
---

New RoboPod [`Charts.framework`](https://github.com/danielgindi/Charts), currently it is in testing. If there is an interest from community will create a pull request to `MobiVM/RoboPods`. Meanwhile it is available from my own `maven` repo for intermediate results `altpods`:

<!-- more -->
`Altpods` are available as repository in [github](https://github.com/dkimitsa/robovm-robopods), also I deploy it to 'oss.sonatype.org' so it is available as maven artifact.

>Charts: Beautiful charts for iOS/tvOS/OSX! The Apple side of the crossplatform MPAndroidChart.

![]({{ "/assets/2017/12/27/ChartsFrameworkDemo.png" | absolute_url}})

> WARNING: as today there is bug in RoboVM that doesn't allow it to run [Issue #244](https://github.com/MobiVM/robovm/issues/244) and [fix PR#245](https://github.com/MobiVM/robovm/pull/245). To early access it custom build of RoboVM with this PR is applied

### to use Charts pod configure your `robovm.xml`

```
<config>
    ...
    <frameworkPaths>
        <path>libs</path>
    </frameworkPaths>
    <frameworks>
        <framework>Charts</framework>
    </frameworks>
</config>
```

### Add the following dependency to your `build.gradle`:

```
dependencies {
   ... other dependencies ...
   compile "io.github.dkimitsa.robovm:robopods-charts-ios:$altpodsVersion"
}
```

Complete sample application is available [by link](https://github.com/dkimitsa/robovm-samples/tree/alt/robopods/charts/ios)  
