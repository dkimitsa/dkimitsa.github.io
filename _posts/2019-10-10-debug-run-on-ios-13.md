---
layout: post
title: "fix414: run/debug on ios13 device"
tags: [fix, ios13, libmobiledevice]
---
**History:** [errors when deploying to ios13 #414](https://github.com/MobiVM/robovm/issues/414)  
**Fix:** [PR416](https://github.com/MobiVM/robovm/pull/416)  

## Root case 
[libmobiledevice](https://github.com/libimobiledevice/libimobiledevice) when locking services has to perform SSL handshake if required.  
in iOS13 debugerserver started requiring this as well by including `EnableServiceSSL` in response:  
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>EnableServiceSSL</key>
	<true/>
	<key>Port</key>
	<integer>49745</integer>
	<key>Request</key>
	<string>StartService</string>
	<key>Service</key>
	<string>com.apple.debugserver</string>
</dict>
```

The weird thing is that after SSL handshake debug server service switches back to plain text communication over same underlayer socket. There was a [try-fix in libmobiledevice](https://github.com/libimobiledevice/libimobiledevice/commit/7c00b369a3665cd0aeae228b2a2866da8ee027fd) to close SSL connection but it didn't work. 
Root case of it is that it was trying close SSL connection by calling `SSL_shutdown` which sends close_notify shutdown to underlaying socket. But debugserver expects clear text GDP protocol communication and `close_notify` aren't expected which causes gdp protocol error.  
This issue was also discussed on `libimobiledevice` repo: [issue789](https://github.com/libimobiledevice/libimobiledevice/issues/793)  

## The fix
<!-- more -->
It is enough just abandon SSL connection without using `SSL_shutdown` and just free all related resources. Corresponding fix was delivered to `libimobiledevice` as [PR860](https://github.com/libimobiledevice/libimobiledevice/pull/860), also [PR859](https://github.com/libimobiledevice/libimobiledevice/pull/859) was opened to deliver required TIMEOUT error code.  
Meanwhile these contributions might not be included in master of `libimobiledevice` same fixes were delivered to MobiVM in form of [patches](https://github.com/dkimitsa/robovm/tree/aa6c09ebb7ecd92362c01220208e992731dabcfc/compiler/libimobiledevice/patches) and included in [PR416](https://github.com/MobiVM/robovm/pull/416)

## Other notes 
Beside the fix itself [PR416](https://github.com/MobiVM/robovm/pull/416) delivers following changes:  
* updated to recent master of `libimobiledevice`
* utilized `debugserver api` instead of service/device communication as it is already part of `libimobiledevice` and any fixes there will be automaticaly adopted;
* `libimobiledevice` bindings were refreshed;
* Method that finds Developer Image to mount was updated to allow using not exact one but matching major version (e.g. to allow mount ios13.0 image on ios13.1 device)

 
