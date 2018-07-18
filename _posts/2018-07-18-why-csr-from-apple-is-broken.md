---
layout: post
title: 'Apple generates broken CertificateSigningRequest.certSigningRequest but its ok (for Apple)'
tags: [bug, apple, codesign, linux-windows]
---

Keep working on Keychain utility for linux-windows port, generating the code signing request for particular. And was surprised that openssl was not able to load it `CertificateSigningRequest.certSigningRequest` built by Apple:
```
$ openssl req -text -noout -verify -in CertificateSigningRequest.certSigningRequest
unable to load X509 request
140735602271176:error:0D07209B:asn1 encoding routines:ASN1_get_object:too long:/BuildRoot/Library/Caches/com.apple.xbs/Sources/libressl/libressl-22.50.2/libressl/crypto/asn1/asn1_lib.c:143:
... other complains ...
140735602271176:error:0906700D:PEM routines:PEM_ASN1_read_bio:ASN1 lib:/BuildRoot/Library/Caches/com.apple.xbs/Sources/libressl/libressl-22.50.2/libressl/crypto/pem/pem_oth.c:84:
```
Apple uses different format ? No, it just writes broken ASN.1 stream. Details bellow:  
<!-- more -->

## Way to reproduce
To create broken CSR just keep `Common Name` empty while creating:  
![]({{ "/assets/2018/07/18/keychain_csr.png" | absolute_url}})

[PR311](https://github.com/MobiVM/robovm/pull/311). Kees van Dieren tried to release app to Apple with [RoboVM 2.3.4]({{ site.baseurl }}{% post_url 2018-06-21-robovm-2-3-4-released-whats-new %}) and it [got rejected](https://gitter.im/MobiVM/robovm?at=5b2bfd8c72b31d3691e508a7). It was crashing due NPE at following:
```java
public DecimalFormatSymbols(Locale locale) {
...
    try {
        currency = Currency.getInstance(locale);
        currencySymbol = currency.getSymbol(locale); // <== here
        intlCurrencySymbol = currency.getCurrencyCode();
    } catch (IllegalArgumentException e) {
...
    }
}
```

And `openssl`, `bouncy castle` and several online services I was trying will complain that it is broken.

## Autopsy
I used custom built `bouncy castle` to find the reason of failure and it pointed to broken `Common Name` ASN.1 object. I've converted simplified version of CSR to binary form and it looks as bellow:  
![]({{ "/assets/2018/07/18/apple_broken_csr.png" | absolute_url}})

Highlighted is `Common name` object in ASN.1 representation:
* 0x30 -- tag, tag no == SEQUENCE
* 0x07 -- size of SEQUENCE, 7 bytes
* 0x06 -- tag, tag no == OBJECT_IDENTIFIER
* 0x03 -- size of OID
* 0x55, 0x04, 0x03 -- OID, 2.5.4.3 == Common Name (CN)
* 0x0C -- tag, tag no == type UTF8 string
* 0x01 -- length of string, 1 byte
* the end

**Problem:**  for empty `Common Name` there is specified string length 1 byte, but this data is missing -- broken. That is `openssl` and others complain for,

## Reproducing at Apple side
To reproduce I've create simple app that uses `Security.framework`. Implementation was taken from [source code released by Apple](https://opensource.apple.com/source/Security/Security-58286.41.2/) :
```c
#include <Security/SecKey.h>
#include <Security/SecItem.h>
#include <Security/SecCertificate.h>
#include <CoreFoundation/CoreFoundation.h>
#include <stdlib.h>
#include <unistd.h>
#include <fcntl.h>

extern const void * kSecOidCommonName;
extern const void * kSecOidCountryName;
CFTypeRef kSecOidEmailAddress = CFSTR("1.2.840.113549.1.9.1");
extern const unsigned char SecASN1PrintableString;
extern const unsigned char SecASN1UTF8String;
typedef struct {
    const void * oid;    // kSecOid constant or CFDataRef with oid
    unsigned char type; // currently only SecASN1PrintableString
    CFTypeRef value;    // CFStringRef -> ASCII, UTF8, CFDataRef -> binary
} SecATV;
typedef SecATV *SecRDN;

extern CFDataRef SecGenerateCertificateRequestWithParameters(SecRDN *subject,
    CFDictionaryRef parameters, SecKeyRef publicKey, SecKeyRef privateKey) CF_RETURNS_RETAINED;

int main(int argc, const char * argv[]) {
    SecKeyRef ecPublicKey = NULL, ecPrivateKey = NULL;
    int keysize = 256;
    CFNumberRef key_size_num = CFNumberCreate(kCFAllocatorDefault, kCFNumberIntType, &keysize);

    const void * keyParamsKeys[] = { kSecAttrKeyType, kSecAttrKeySizeInBits };
    const void * keyParamsValues[] = { kSecAttrKeyTypeECSECPrimeRandom,  key_size_num};
    CFDictionaryRef keyParameters = CFDictionaryCreate(NULL, keyParamsKeys, keyParamsValues, 2,
                                                       &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    SecKeyGeneratePair(keyParameters, &ecPublicKey, &ecPrivateKey);

    SecATV e[]  = { {kSecOidEmailAddress, SecASN1UTF8String, CFSTR("user2@google.com") }, {} };
    SecATV cn[]  = { { kSecOidCommonName, SecASN1UTF8String, CFSTR("\0") }, {} };
    SecATV c[]  = { { kSecOidCountryName, SecASN1UTF8String, CFSTR("US") }, {} };
    SecRDN atvs_phone[] = { e, cn, c, NULL };

    CFDataRef csr = SecGenerateCertificateRequestWithParameters(atvs_phone, NULL, ecPublicKey, ecPrivateKey);
    FILE * f = fopen("apple.bin2", "wb");
    fwrite(CFDataGetBytePtr(csr), 1, CFDataGetLength(csr), f);
    fclose(f);
    return 0;
}
```

And it creates same broken `CSR` so the issue in `Security.framework`

## Going deeper into `Security.framework`
Build frameworks from sources provided by apple it is another long story (imposible?) as it depends on close source binaries that are missing. To make it simple just ripped out code I was interesting it to debug trought it. It was [OSX/sec/Security/SecCertificateRequest.c](https://opensource.apple.com/source/Security/Security-58286.41.2/OSX/sec/Security/SecCertificateRequest.c.auto.html) and [OSX/libsecurity_asn1](https://opensource.apple.com/source/Security/Security-58286.41.2/OSX/libsecurity_asn1/) and integrated in small and dirty project just for debugging sake.   

**Long story short**: the reason why for zero length string it was putting 1 byte length is following code:
```
static unsigned long
sec_asn1e_contents_length (const SecAsn1Template *theTemplate, void *src,
			   PRBool ignoresubstream, PRBool *noheaderp)
{
...
  switch (underlying_kind) {
...
    default:
    len = ((SecAsn1Item * )src)->Length;
    if (may_stream && len == 0 && !ignoresubstream)
      len = 1;	// if we're streaming, we may have a secitem w/len 0 as placeholder
      break;
  }      
...
return len;
```

Switch above finds out length for several explicit types, but in case of string it falls down to default case and for zero length payload it just adds fuzzy logic about size and breaks everything.  
**Solution**: Add explicit string case and return proper size:
```
static unsigned long
sec_asn1e_contents_length (const SecAsn1Template *theTemplate, void *src,
			   PRBool ignoresubstream, PRBool *noheaderp)
{
...
  switch (underlying_kind) {
...
    case SEC_ASN1_UTF8_STRING:
    case SEC_ASN1_PRINTABLE_STRING:
        /** probably other kind of strings should be considered as explicit buffer and size returned as it is */
        len = ((SecAsn1Item *)src)->Length;
        break;

    default:
        len = ((SecAsn1Item * )src)->Length;
        if (may_stream && len == 0 && !ignoresubstream)
          len = 1;	// if we're streaming, we may have a secitem w/len 0 as placeholder
        break;
  }      
...
return len;
```

## Fix to be provided by Apple
Meanwhile I've created [ticket]() at Apple's bug tracker.
