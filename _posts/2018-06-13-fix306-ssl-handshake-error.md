---
layout: post
title: 'bugfix #306: enabling TLSv1.2 support'
tags: [fix, debugger, kotlin]
---
**History** [SSL Handshake error #306](https://github.com/MobiVM/robovm/issues/306)  
**Fix** [PR308](https://github.com/MobiVM/robovm/pull/308)  

Initially I was suspecting missing of CA certificate for domain as in [this issue](https://github.com/MobiVM/robovm/pull/167) but it turned out that RoboVM RT was not able to setup connection with `TLSv1.2 only` server.
<!-- more -->
## Wiresharking and first fix
Sniffer had shown that RoboVM is trying to establish TLSv1.0 connection and it is being rejected by server due invalid version. Quick fix is to add `TLSv1.2` to list of default protocols:
```
public final class NativeCrypto {
    public static String[] getDefaultProtocols() {
        return new String[] { SUPPORTED_PROTOCOL_SSLV3,
                              SUPPORTED_PROTOCOL_TLSV1,
                              SUPPORTED_PROTOCOL_TLSV1_2 };
}
```
But it turned out that OpenSSL tries to establish TLSv1.0 connection and completely ignores TLSv1.2 if it set altogether with TLSv1.0 and SSLv3. This was solved by leaving only TLSv1.2 in the list.
Why this is working:
* TLSv1.2 is backward compatible with 1.0 and 1.1 and will not break TLS case;
* there is fallback for SSLv3 in RoboVM which will be triggered in case TLS is not working (so no need to include SSLv3 in the list at all);  
Final snippet:  

```
public final class NativeCrypto {
    public static String[] getDefaultProtocols() {
        return new String[] { SUPPORTED_PROTOCOL_TLSV1_2};
    }
}
```

It doesn't complains on version anymore but does't work neither

## Missing cipher suites
RoboVM's setup was missing few ciphers that are required for TLSv1.2 and these were the result of failure. Adding following solves TLSv1.2 connection:
```
"TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
"TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
"TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384",
"TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256",
```

## Other moments
RoboVM's SSL implementation is limited in way it doesn't honor SSLParameters obtained from SSLContext. In other words following code will have no affect:
```
SSLContext ctx = SSLContext.getInstance("TLSv1.2");
ctx.init(null, null, null);
SSLContext.setDefault(ctx);
```

or this:
```
HttpsURLConnection req = (HttpsURLConnection) url.openConnection();

SSLContext ctx = SSLContext.getInstance("TLSv1.2");
ctx.init(null, null, null);
req.setSSLSocketFactory(ctx.getSocketFactory());
```
