---
layout: post
title: 'RoboPods: Airbnb Lottie framework'
tags: ['bro-gen', binding, robopods, altpods]
date: 2018-03-09 13:04:00

---

New [`Lottie.framework`](https://github.com/airbnb/lottie-ios) pod. It is nice and shiny library for  natively render After Effects vector animations, like this:   
![]({{ "/assets/2018/03/09/pin-jump.gif" | absolute_url}})  

RoboPod is available from my own `maven` repo for intermediate results `altpods`:   
`Altpods` are available as repository in [github](https://github.com/dkimitsa/robovm-robopods), also it is deployed it to 'oss.sonatype.org' which makes it available as maven artifact.

<!-- more -->
### to use Lottie pod configure your `robovm.xml`

```
<config>
    ...
    <frameworkPaths>
        <path>libs</path>
    </frameworkPaths>
    <frameworks>
        <framework>Lottie</framework>
    </frameworks>
</config>
```

### Add the following dependency to your `build.gradle`:

```
dependencies {
   ... other dependencies ...
    compile "io.github.dkimitsa.robovm:robopods-lottie-ios:$altpodsVersion"
}
```

Sample application is available [by link](https://github.com/dkimitsa/robovm-samples/tree/alt/robopods/lottie/ios)  
