---
layout: post
title: 'Eldest bugs: Memory corruption during HTTPS operations'
tags: [fix, runtime, okhttp]
---
```
error:14077102:SSL routines:SSL23_GET_SERVER_HELLO:unsupported protocol 
```
This error appeared time to time in issues ([#601](https://github.com/MobiVM/robovm/issues/601) [#585](https://github.com/MobiVM/robovm/issues/585) [#557](https://github.com/MobiVM/robovm/issues/557) [#306](https://github.com/MobiVM/robovm/issues/306)) and gitter. 
Most of the time it was about crash reports without sample to reproduce.  
And here Tom-Ski brings reproducible case...

Lets investigate. 
<!-- more -->

## Symptoms 
First of all lets check the place where error is happening:
![]({{ "/assets/2024/05/08/unsupported-protocol-stack.png"}})

OpenSSL is not happy with `Server Hello` packet. Let's look into `p` vector values:
```
[0]	char	'\x15' SSL3_RT_ALERT
[1]	char	'\x03' TLS1_VERSION_MAJOR
[2]	char	'\x01' TLS1_VERSION_MINOR
[3]	char	'\0'
[4]	char	'\x02'
[5]	char	'\x02'
[6]	char	'F'    TLS1_AD_PROTOCOL_VERSION 
[7]	char	'\0'
```

Fatal `TLS1_AD_PROTOCOL_VERSION` tells us that requested protocol version TLS1 was rejected. 
Here it goes in Wireshark:

![]({{ "/assets/2024/05/08/unsupported-protocol-wireshark.png"}})


While it is quite weird RoboVM requested TLS1 as its deprecated almost everywhere lets check wireshark for other errors. Here comes another interesting Alert:

![]({{ "/assets/2024/05/08/unexpected-message-wireshark.png"}})


This time Fatal alert comes from RoboVM side, means it is not happy with `Server Hello`. Break point on `ssl3_send_alert` shows where it happens:

![]({{ "/assets/2024/05/08/unsupported-message-stacktrace.png"}})


And it happens as there was no Certificate in `Server Hello` but it is not expected, as its `New Session Ticket` response:
> Server Hello, New Session Ticket, Change Cipher Spec, Encrypted Handshake Message

This means RoboVM code initiated handshake with [rfc4507 session ticket](https://datatracker.ietf.org/doc/html/rfc4507) to resume session but started looking for certificates in response.  
Going up stack track while staying on breakpoint shows following:
```cpp
		case SSL3_ST_CR_CERT_A:
		case SSL3_ST_CR_CERT_B:
#ifndef OPENSSL_NO_TLSEXT
			ret=ssl3_check_finished(s);
			if (ret <= 0) goto end;
			if (ret == 2)
				{
				s->hit = 1;
				if (s->tlsext_ticket_expected)
					s->state=SSL3_ST_CR_SESSION_TICKET_A;
				else
					s->state=SSL3_ST_CR_FINISHED_A;
				s->init_num=0;
				break;
				}
#endif
			/* Check if it is anon DH/ECDH */
			/* or PSK */
			if (!(s->s3->tmp.new_cipher->algorithm_auth & SSL_aNULL) &&
			    !(s->s3->tmp.new_cipher->algorithm_mkey & SSL_kPSK))
				{
				ret=ssl3_get_server_certificate(s);
```

It means `ssl3_check_finished` fails, and it indeed had returned `1` and code for it following:
```cpp
int ssl3_check_finished(SSL *s)
	{
	int ok;
	long n;
	/* If we have no ticket it cannot be a resumed session. */
	if (!s->session->tlsext_tick)
		return 1;
	.... 
	}
```

It seems like session ticked is missing. That is completely strange as at moment of sending `Client Hello` it was present and included into the packet.

It would take some time to investigate why session got corrupted by looking through all kind of synchronization but lucky for it crashed at exact place I was looking:
![]({{ "/assets/2024/05/08/sigabrt-on-dealloc.png"}})

Releasing not valid pointer.

# Root case 
Long story short -- there few places where session object and considering that its related to Session resume using session tickets this quick lead to Java side.

## Shared session
Shared sessions are stored in HashMap in `ClientSessionContext` with host+port as a key. Java object for session contains native pointer to OpenSSL corresponding struct (sslSessionNativePointer). 

Shared session is picked by `OpenSSLSocketImpl` during handshake:
```java
OpenSSLSessionImpl sessionToReuse;
    if (client) {
        ClientSessionContext clientSessionContext = this.sslParameters.getClientSessionContext();
        sessionContext = clientSessionContext;
        sessionToReuse = this.getCachedClientSession(clientSessionContext);
        if (sessionToReuse != null) {
            NativeCrypto.SSL_set_session(this.sslNativePointer, sessionToReuse.sslSessionNativePointer);
        }
    }
```
It just sets it as pointer of cached session into `SSL` on native side (it is not copied and used as it is).
`OpenSSLSessionImpl` doesn't perform any more action with `sessionToReuse`. 

But when socket connection is closed, it deallocates own `SSL` native object:
```java
 private void free() {
     if (this.sslNativePointer != 0L) {
        NativeCrypto.SSL_free(this.sslNativePointer);
        this.sslNativePointer = 0L;
        this.guard.close();
    }
}
```

And `SSL_free` destroys SHARED session:
```cpp
void SSL_free(SSL *s)
	{
....
	/* Make the next call work :-) */
	if (s->session != NULL)
		{
		ssl_clear_bad_session(s);
		SSL_SESSION_free(s->session);
		}
```

As result:
- `ClientSessionContext` keeps session object with pointer that was already de-allocated at native side;
- next attempt to use `sessionToReuse` will operate with de-allocated memory;
- closing `OpenSSLSessionImpl` with such session will de-allocated already invalid pointer. 

# the "fix"
Existing Conscrypt and OpenSSL code used in RoboVM is at Android4.4 level and pretty old.  
Trying to fix it on existing source seems not easy/possible. Also, recent Android/Libcore handle these things completely different way. 
There are moments present that are not taken in account: session might be marked as `not_resumable` and can't be used in existing logic. 

All this makes quite simple option -- disable session caching. 
It will affect `https` handshake as subsequent connections to same port/host will require full handshake with certificate exchange:

With session caching:
![]({{ "/assets/2024/05/08/hello-with-caching.png"}})

And without:
![]({{ "/assets/2024/05/08/hellow-without-caching.png"}})

# Code
Fix was proposed as [PR785](https://github.com/MobiVM/robovm/pull/785)

PS: There is an options to go back to `cached` way by defining system property:
> System.setProperty("SSL_FAULTY_RFC4507_ON", "");
