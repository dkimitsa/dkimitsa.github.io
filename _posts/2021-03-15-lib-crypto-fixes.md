---
layout: post
title: 'fix #557: Applying fixes to libcore'
tags: ['fix']
---

## Background:
[issue #557](http request memory corruption crash during SSL handshake ) http request memory corruption crash during SSL handshake.  
Simple test case indeed caused the crash, code to reproduce is following:   
<!-- more -->  
```java
 private void onClick() {
      for (int i = 0; i < 30; i++) {
          new Thread(() -> {
              try {
                  URL url = new URL("https://google.com");
                  final HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                  connection.setDoOutput(true);
                  connection.setDoInput(true);
                  connection.setRequestMethod("POST");
                  HttpURLConnection.setFollowRedirects(true);
                  OutputStream os = connection.getOutputStream();
                  os.close();
                  connection.disconnect();
              } catch (Throwable t) {
                  t.printStackTrace();
              }
          }).start();
      }
  }
```
it causes crash inside `OpenSSL`:  
```
Application Specific Information:
abort() called
Main(62669,0x7000058fa000) malloc: *** error for object 0x7fe77743ed70: pointer being freed was not allocated
Thread 31 Crashed:
0   __pthread_kill + 10
1   pthread_kill + 263
2   abort + 120
3   malloc_vreport + 548
4   malloc_report + 151
5   CRYPTO_free + 54 (mem.c:397)
6   ssl_parse_serverhello_tlsext + 807 (t1_lib.c:1598)
7   ssl3_get_server_hello + 2299 (s3_clnt.c:1124)
8   ssl3_connect + 1307 (s3_clnt.c:312)
9   SSL_do_handshake + 178 (ssl_lib.c:2692)
10  Java_com_android_org_conscrypt_NativeCrypto_SSL_1do_1handshake + 667 (org_conscrypt_NativeCrypto.cpp:7109)
11  [J]com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(JLjava/io/FileDescriptor;Lcom/android/org/conscrypt/NativeCrypto$SSLHandshakeCallbacks;IZ[B[B)J + 141
12  [j]com.android.org.conscrypt.NativeCrypto.SSL_do_handshake(JLjava/io/FileDescriptor;Lcom/android/org/conscrypt/NativeCrypto$SSLHandshakeCallbacks;IZ[B[B)J[clinit] + 114
```

Code under `t1_lib.c:1598` looks like bellow: 
```c
1598:  if (s->session->tlsext_ellipticcurvelist != NULL) OPENSSL_free(s->session->tlsext_ellipticcurvelist);
1599:  if ((s->session->tlsext_ellipticcurvelist = OPENSSL_malloc(ellipticcurvelist_length)) == NULL)
```

## Root case
When invoking request to single domain `Libcore` reuses session and all corresponding `SSL` instances point to same object. As result there is a chance that two handshakes will perform `OPENSSL_freeOPENSSL_free(s->session->tlsext_ellipticcurvelist)` same time due race conditions.   
One of the possible fixes is to disable the session caching (and it helps).  
Checking the web show several fixes old `Android 4.4` Crypto suffers:

## Fix1: 89408: ALPN: change socket calls to SSL_set_alpn_protos
> Calling SSL_CTX_set_alpn_protos appears to be detrimental to thread safety since the implementation of it resets the values. It's not idempotent to call it multiple times like SSL_CTX_enable_npn.

Widely discussed on [okhttp](https://github.com/square/okhttp/issues/666) issue tracked.   
Fixes were merged from Android [PR-89408](https://android-review.googlesource.com/c/platform/external/conscrypt/+/89408) but it doesn't solve the issue. 
[Commit](https://github.com/dkimitsa/robovm/commit/149d6dadf8408a2af53794e6b7ddbe9106bc109f)

## Fix2: 68360: Tidy up locking in OpenSSLSocketImpl
> We guard all state with a single lock "stateLock", which
replaces usages of "this" and "handshakeLock". We do not
perform any blocking operations while holding this lock.
In particular, startHandshake is no longer synchronized.
We use a single integer to keep track of handshake state
instead of a pair of booleans.
Also fix a bug in getSession, the previous implementation
wouldn't work in cut-through mode.

Up to google guys in comments for [issue](https://issuetracker.google.com/issues/37002161) these changes should fix the issue we have. But it doesn't.
[PR-68360](https://android-review.googlesource.com/c/platform/external/conscrypt/+/68360) seems to be useful so merged as well. [Commit](https://github.com/dkimitsa/robovm/commit/3c6e25346890489ee9240481e554bd46a7f8e2af)

## Fix3: CVE-2010-3864: TLS extension parsing race condition
> A flaw has been found in the OpenSSL TLS server extension code parsing which
on affected servers can be exploited in a buffer overrun attack.  
[link](https://www.openssl.org/news/secadv/20101116-2.txt)  

Strange thing that `OpenSSL` used withing old Libcore contains part of the fixes, but the section that caused a crash.
Applying missing code fixes original issue (hooray!). [Commit](https://github.com/dkimitsa/robovm/commit/4ce0a23fdda1598531ec06767dd91fe2377e3b23)

## NB:
It's one of reason why we should finish and migrate as soon as possible to updated [Libcore10]({{ site.baseurl }}{% post_url 2020-10-06-runtime-update-libcore10 %}).

## Code 
Changes proposed as [PR564](https://github.com/MobiVM/robovm/pull/564)
