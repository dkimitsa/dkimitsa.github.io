---
layout: post
title: 'okhttp 4x: failed to connect to cloudflare protected server'
tags: [fix, okhttp]
---
[okhttp](https://github.com/square/okhttp) stopped working again with RoboVM: once project migrated to kotlin and refactored code [old issue is back]({{ site.baseurl }}{% post_url 2018-09-11-robovm-and-okhttp3-3-11-0 %}):  
> java.lang.NoClassDefFoundError: android/os/Build$VERSION 

Workaround is same:  
```java
package android.os;
public class Build {
    public final static class VERSION {
        public final static int SDK_INT = 21;
    }
}
```  

Worse thing is that `okhttp` unable to connect to cloudflare protected servers:  
```
javax.net.ssl.SSLHandshakeException: javax.net.ssl.SSLProtocolException: SSL handshake terminated: ssl=0x7fae5b122560: Failure in SSL library, usually a protocol error
error:14094410:SSL routines:SSL3_READ_BYTES:sslv3 alert handshake failure
```  

While old good `URL u = new URL(url); u.openConnection();` works. 

## Root case
Usually handshake failed when parameter of connection failed to negotiate. Good tool for investigating such things is a sniffer like Wireshark. Longs story short -- it fails as its not able to negotiate a cipher to be used for connection. Cloudflare supports following ones:     
<!-- more -->

* TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 
* OLD_TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
* TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384  

Ciphers RoboVM supports is listed in `NativeCrypto.getSupportedCipherSuites` and two from this list matches server ones (both are weak today):
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA

This expected as `url.openConnection()` works.  
Back to `okhttp`. It has own `ConnectionSpec.APPROVED_CIPHER_SUITES` that lists following ciphers:  
```
// TLSv1.3.
CipherSuite.TLS_AES_128_GCM_SHA256,
CipherSuite.TLS_AES_256_GCM_SHA384,
CipherSuite.TLS_CHACHA20_POLY1305_SHA256,

// TLSv1.0, TLSv1.1, TLSv1.2.
CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,
CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,
CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384,
CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384,
CipherSuite.TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256,
CipherSuite.TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256,

// Note that the following cipher suites are all on HTTP/2's bad cipher suites list. We'll
// continue to include them until better suites are commonly available.
CipherSuite.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA,
CipherSuite.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA,
CipherSuite.TLS_RSA_WITH_AES_128_GCM_SHA256,
CipherSuite.TLS_RSA_WITH_AES_256_GCM_SHA384,
CipherSuite.TLS_RSA_WITH_AES_128_CBC_SHA,
CipherSuite.TLS_RSA_WITH_AES_256_CBC_SHA,
CipherSuite.TLS_RSA_WITH_3DES_EDE_CBC_SHA
```

During operations `okhttp` intersects own `APPROVED_CIPHER_SUITES` with `NativeCrypto.getSupportedCipherSuites` to get the list of approved ciphers available in system. This gives the list:  
 * TLS_RSA_WITH_AES_128_CBC_SHA
 * TLS_RSA_WITH_AES_256_CBC_SHA
 * TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
 * TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
 * TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
 * TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
 * SSL_RSA_WITH_3DES_EDE_CBC_SHA

None of these are compatible with Cloudflare and handshake error is expected.

## Instant workaround
One that works out of box -- create own connection spec with missing cipher and use it to create the client:   
```java
List<CipherSuite> roboSuite = new ArrayList<>(ConnectionSpec.COMPATIBLE_TLS.cipherSuites());
roboSuite.add(CipherSuite.TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA);
ConnectionSpec roboSpec = new ConnectionSpec.Builder(true)
        .cipherSuites(roboSuite.toArray(new CipherSuite[0]))
        .tlsVersions(TlsVersion.TLS_1_3, TlsVersion.TLS_1_2, TlsVersion.TLS_1_1, TlsVersion.TLS_1_0)
        .supportsTlsExtensions(true)
        .build();
OkHttpClient client = new OkHttpClient.Builder()
        .connectionSpecs(Collections.singletonList(roboSpec))
        .build();

``` 

This will not help if problem happens in third party dependency code.  

## Better fix 
Event outdated openssl we use for RoboVM has more ciphers than currently used. The fix is to get as maximum as possible and put it into outdated Android 4 runtime.  
The only moment -- no need to resurrect old and outdated ones. Will use JDK13 to get modern list and try to get ones openssl. Matching different ids with script unlocked following ciphers:  
```
/* TLS v1.2 ciphersuites */
TLS_RSA_WITH_AES_256_CBC_SHA256
TLS_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_RSA_WITH_AES_256_CBC_SHA256
TLS_DHE_DSS_WITH_AES_256_CBC_SHA256
TLS_DHE_RSA_WITH_AES_128_CBC_SHA256
TLS_DHE_DSS_WITH_AES_128_CBC_SHA256

/* TLS v1.2 GCM ciphersuites from RFC5288 */
TLS_DHE_RSA_WITH_AES_256_GCM_SHA384
TLS_DHE_DSS_WITH_AES_256_GCM_SHA384
TLS_DHE_RSA_WITH_AES_128_GCM_SHA256
TLS_DHE_DSS_WITH_AES_128_GCM_SHA256
TLS_RSA_WITH_AES_256_GCM_SHA384
TLS_RSA_WITH_AES_128_GCM_SHA256

/* ECDH HMAC based ciphersuites from RFC5289 */
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDH_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDH_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDH_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDH_RSA_WITH_AES_128_CBC_SHA256

/* ECDH GCM based ciphersuites from RFC5289 */
TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDH_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDH_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDH_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDH_RSA_WITH_AES_128_GCM_SHA256
``` 

This give following ciphers compatible with system URLLoader:
* TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
* TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384  

And two compatible with both `okhttp` and Cloudflare:
* TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 
* TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384

## Other changes
* list of supported ciphers was re-ordered to match order of JDK13;
* list of default cipher was extended with new ones and re-ordered to match order of JDK13. 

## Code 
Delivered as [PR496](https://github.com/MobiVM/robovm/pull/496).

## NP
Update to recent Java runtime is still required. One day old code will just stop working. 
  
 
