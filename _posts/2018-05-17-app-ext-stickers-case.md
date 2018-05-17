---
layout: post
title: 'Fix for Stickers AppExtensions: fails to complete Apple Store validation'
tags: [fix, 'app-extensions', ]
---
Event after fixing [signature case]({{ site.baseurl }}{% post_url 2018-05-16-fix-app-ext-autoprofile-bro-gen %}) apple keeps rejecting applications with Stickers app extension with message:
>Invalid iMessage App - Your iMessage app contains an invalid sticker pack. The app may have been built or signed with non-compliant or prerelease tools. For more information, go to developer.apple.com.

[user @sofroma investigated case](https://gitter.im/MobiVM/robovm?at=5afd1f862df44c2d063453b0) and has found that Xcode additionally copies support files into Apple store package. The package was missing following files:
* MessagesApplicationExtensionSupport/MessagesApplicationExtensionStub
* MessagesApplicationSupport/MessagesApplicationStub

These files are part of iPhoneOS platform and available as part of Xcode at following location: `/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Library/Application Support`

The fix is to copy these file during packing `ipa`, check this [commit](https://github.com/MobiVM/robovm/pull/293/commits/9d7b0bceeeb5db9485898ebab317728c035df5b7) for details.  
Sadly these files are part of XCode and can't be used on Windows/Linux in [corresponding RoboVM port]({{ site.baseurl }}{% post_url 2017-12-12-robovm-now-linux-windows %})
