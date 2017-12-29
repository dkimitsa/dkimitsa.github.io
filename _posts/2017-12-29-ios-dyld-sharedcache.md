---
layout: post
title: 'iOS: de-caching dyld shared cache and hidden gem'
tags: [hacking]
---
Since iOS3 all system dylib/frameworks are combined into big cache file for performance reason. I needed these files to obtain export symbols from them for my future work on Windows/Linux support for RoboVM. There are several ways to get these.  
<!-- more -->
There is lot of information about caches in [iPhone Development Wiki](http://iphonedevwiki.net/index.php/Dyld_shared_cache)  
First of all cache files are located in `/System/Library/Caches/com.apple.dyld/`. There are two files: `dyld_shared_cache_armv7s.dms` and `dyld_shared_cache_arm64.dms`.  

### Retrieving cache
There are several way to extract these files listed in [iPhone Development Wiki](http://iphonedevwiki.net/index.php/Dyld_shared_cache), but lets go `RoboVM` way and use all power of `java/maven repos`. This gives us simple project to run. Once it is started there will be file web server on device, navigate your desktop browser to `http://x.x.x.x:8081` (x.x.x.x is IP address of your iOS device) and files are available there to download.
```java
// app delegate class
@Override
public boolean didFinishLaunching(UIApplication application, UIApplicationLaunchOptions launchOptions) {
    ...
    new Thread(() -> {
        try {
            SimpleWebServer server = new SimpleWebServer(null, 8081, new File("/System/Library/Caches/com.apple.dyld/"), false);
            server.start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }).start();

    return true;
}
```

```gradle
// build.gradle
dependencies {
    ...
    // https://mvnrepository.com/artifact/org.nanohttpd/nanohttpd-webserver
    compile group: 'org.nanohttpd', name: 'nanohttpd-webserver', version: '2.3.1'
}
```

### Dealing with cache file
There is bunch of tools listed in [iPhone Development Wiki](http://iphonedevwiki.net/index.php/Dyld_shared_cache):
* [dyld_decache](https://github.com/kennytm/Miscellaneous/downloads)
* [DySlim](https://gist.github.com/455086/)
* [decache](https://github.com/phoenix3200/decache)
* [dsc_extractor (from apple)](http://opensource.apple.com/source/dyld/)
* and other

All of them are kind of broken, limited and don't produce reliable result. I started fixing `dyld_decache` but ended in writing own cache parser in java. But suddenly found hidden gems...

### Hidden gems
check [dsc_extractor (from apple)](http://opensource.apple.com/source/dyld/), look at the [end of dsc_extractor.cpp](https://github.com/Apple-FOSS-Mirror/dyld/blob/b608d83fdddaa9b134865007b04f821c6f7b244a/launch-cache/dsc_extractor.cpp#L459) there will be commented out code:
```cpp
#if 0
typedef int (*extractor_proc)(const char* shared_cache_file_path, const char* extraction_root_path,
													void (^progress)(unsigned current, unsigned total));
int main(int argc, const char* argv[])
{
	if ( argc != 3 ) {
		fprintf(stderr, "usage: dsc_extractor <path-to-cache-file> <path-to-device-dir>\n");
		return 1;
	}

	void* handle = dlopen("/Developer/Platforms/iPhoneOS.platform/usr/lib/dsc_extractor.bundle", RTLD_LAZY);
	if ( handle == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle could not be loaded\n");
		return 1;
	}

	extractor_proc proc = (extractor_proc)dlsym(handle, "dyld_shared_cache_extract_dylibs_progress");
	if ( proc == NULL ) {
		fprintf(stderr, "dsc_extractor.bundle did not have dyld_shared_cache_extract_dylibs_progress symbol\n");
		return 1;
	}

	int result = (*proc)(argv[1], argv[2], ^(unsigned c, unsigned total) { printf("%d/%d\n", c, total); } );
	fprintf(stderr, "dyld_shared_cache_extract_dylibs_progress() => %d\n", result);
	return 0;
}
#endif
```
Just paste snippet into .c file, compile it and (if you have installed xcode) it will give you native apple cache extractor.

### Hidden gems 2
It is listed at [iPhone Development Wiki](http://iphonedevwiki.net/index.php/Dyld_shared_cache) but it should be listed with huge text size. Once device is connected xcode is `enabling` device for development. Same time it extracts all dylib with dsc_extractor and puts them to `~/Library/Developer/Xcode/iOS DeviceSupport/`. So just connect device and xcode will extracts everything without any tool. But there will just one arch, e.g. ARM7
