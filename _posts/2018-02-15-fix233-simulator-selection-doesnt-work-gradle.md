---
layout: post
title: 'bugfix #233: Unable to start 12.5 inch iPad simulator'
tags: [fix, gradle]
---
**History** [Unable to start 12.5 inch iPad simulator #233](https://github.com/MobiVM/robovm/issues/233)  
**Fix** [PR273](https://github.com/MobiVM/robovm/pull/273)  

Recently there was a fix for [simulator selection in Idea]({{ site.baseurl }}{% post_url 2018-02-07-fix263-simulator-selection-doesnt-work-idea %}). Its turned out that there is a similar gradle one. But gradle is broken in different way:  
`gradle -Probovm.arch=x86_64 -Probovm.device.name="iPad Pro (12.9-inch) (2nd generation)" launchIPadSimulator`

failed with message  
`Unable to find a matching device [arch=x86_64, family=iPad, name=iPad Pro (12.9-inch) (2nd generation), sdk=null]`

Root case is very simple, code in `DeviceType.filter()` replaced any '-' with ' ' by following code snipped:  
`deviceName = deviceName == null ? null : deviceName.toLowerCase().replaceAll("-", " ");`

as result it was looking for device with name `iPad Pro (12.9 inch) (2nd generation)` (without '-') and of course was not able to locate it.
