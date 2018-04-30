---
layout: post
title: 'bugfix #286, #285 and clear cache on new snapshot logic'
tags: [fix]
---
Morning maintenance [PR289](https://github.com/MobiVM/robovm/pull/289), there are three commits:  

1. fixes [#286 Apps won't run on simulator if embedded frameworks are present](https://github.com/MobiVM/robovm/pull/286) by completing [#289](https://github.com/MobiVM/robovm/pull/289) in sake of delivering the fix
2. fixes [#285 Install failure disables run button](https://github.com/MobiVM/robovm/pull/285) -- obeys not @NotNull otherwise idea halt the plugin
3. adds logic not extracting same file on every launch and do clear cache when new files extracted just to avoid bug cases when incompatible data stays in cache, [check post by @dthommes](https://gitter.im/MobiVM/robovm?at=5ae44a7415c9b0311445a395)
