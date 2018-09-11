---
layout: post
title: 'okhttp3 v3.11.0: playing nice with RoboVM'
tags: [workaround]
---

As stated in [#325](https://github.com/MobiVM/robovm/issues/325) it doesn't work. The reason for this is that `okhttp3` detects RoboVM as Android (as there are corresponding providers classes present). And crashes while tries to access Android's `Build` class, which is not found:  
```
java.lang.NoClassDefFoundError: android/os/Build$VERSION
	at okhttp3.internal.platform.AndroidPlatform.getSSLContext(AndroidPlatform.java:434)
	at okhttp3.OkHttpClient.newSslSocketFactory(OkHttpClient.java:282)
```

Dirty and fast solution is just to add `Build` class to your class path:
```
package android.os;

public final class Build {
    public final static class VERSION {
        public final static int SDK_INT = 16; // android 4.1
    }
}
```


Long term solution would be updating to OpenJDK10.
