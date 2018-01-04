---
layout: post
title: 'Tutorial: iOS - creating self-signed development certificate and provisioning profile to sign intermediate binaries'
tags: [hacking, tutorial, codesign]
---
It is quite rare hacking as there is no problem receive certificate/provisioning profile from Apple using developer account or free provisioning. But there are moments when binary has to be signed during build time(kind of showstopper). In case when the binary is going to be shared with community it would be nice to keep personal account details not disclosed(these are included as part of signature). These cases are: binary distribution of `Frameworks`, `plugins` or `application extensions`. This post describes how to do self-signed provisioning and allow Xcode to use it.  
<!-- more -->
# Signing certificate - just use `Keychain access`  
* select `login` Keychain
* navigate menu `Keychain Access` - `Certificate Assistant` and select `Create certificate...`
* in dialog that appeared, fill following fields:
    - *Name*: `iPhone Developer: Self Signer` or whatever, but *important* to keep `iPhone Developer:` at beginning
    - *Identity type*: keep `Self signed root`
    - *Certificate type*: `Code signing`
    - be sure to check `let me override defaults`  

![]({{ "/assets/2018/01/04/cert-assistant1.png" | absolute_url}})   
* hit continue and in next screen update validity period to *3652* this will to extend it live to 10 years
* next `personal information`: *important* to specify `Organization unit` here as it will be used as `Team identifier`

![]({{ "/assets/2018/01/04/cert-assistant2.png" | absolute_url}})   
* next `Key Pair information` also not required any changes, keep default `Key Size: 2048 bits` and `Algorithm: RSA`
* next `Key Usage Extension` please check that `Signature` is selected
* next `Extended Key Usage Extension` please select beside `Code Signing` additional `Any` and `Email Protection` options. As this certificate will be also used to sign provisioning profile
* keep `Basic Constraints Extension` as it is
* keep `Subject Alternative Name Extension` as it is
* make sure `login` keychain is selected at last screen and hit `create`

# Provisioning profile from scratch  
Some time ago provisioning profiles were just plain plist files but since then and now it is `Cryptographic Message Syntax (CMS)` with plist embedded as payload. Apple signs it when creating with own credentials but it is verified only once it gets to device. plist file can be extracted from provisioning profile with following command line:   
`security cms -D -i angrybirds.mobileprovision`

## Creating plist file
I just dumped plist from random Development provisioning profile and cleaned it into following template:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>AppIDName</key>
	<string>selfsigned: any app</string>
	<key>ApplicationIdentifierPrefix</key>
	<array>
	<string>SELFSIGNED</string>
	</array>
	<key>CreationDate</key>
	<date>2018-01-01T13:00:00Z</date>
	<key>Platform</key>
	<array>
		<string>iOS</string>
	</array>
	<key>DeveloperCertificates</key>
	<array>
		<data>*** CERT WILL GO HERE ***</data>
	</array>
	<key>Entitlements</key>
	<dict>
		<key>keychain-access-groups</key>
		<array>
			<string>SELFSIGNED.*</string>
		</array>
		<key>get-task-allow</key>
		<false/>
		<key>application-identifier</key>
		<string>SELFSIGNED.*</string>
		<key>com.apple.developer.associated-domains</key>
		<string>*</string>
		<key>com.apple.developer.team-identifier</key>
		<string>SELFSIGNED</string>
		<key>aps-environment</key>
		<string>production</string>
	</dict>
	<key>ExpirationDate</key>
	<date>2028-01-01T14:00:00Z</date>
	<key>Name</key>
	<string>DO NOT USE: only for dummy signing</string>
	<key>ProvisionedDevices</key>
	<array>
	</array>
	<key>TeamIdentifier</key>
	<array>
		<string>SELFSIGNED</string>
	</array>
	<key>TeamName</key>
	<string>Selfsigners united</string>
	<key>TimeToLive</key>
	<integer>3652</integer>
	<key>UUID</key>
	<string>B5C2906D-D6EE-476E-AF17-D99AE14644AA</string>
	<key>Version</key>
	<integer>1</integer>
</dict>
</plist>
```

## Filling with valid data
*required*: `date constraints` -- just use same values as in certificate (can be checked in Keychain Access):

```xml
<key>CreationDate</key>
<date>2018-01-01T13:00:00Z</date>
<key>ExpirationDate</key>
<date>2028-01-01T14:00:00Z</date>
<key>TimeToLive</key>
<integer>3652</integer>
```
*required*: 'DeveloperCertificates'. Pick the certificate value from Keychain with following command line:
`security find-certificate -c "iOS Development: Self Signer" -p`  
it will print certificate in `.pem` format:

```
-----BEGIN CERTIFICATE-----
MIIDgzCCAmugAwIBAgIBATANBgkqhkiG9w0BAQsFADBnMSYwJAYDVQQDDB1pUGhv
bmUgRGV2ZWxvcGVyOiBTZWxmIFNpZ25lcjEbMBkGA1UECgwSU2VsZnNpZ25lcnMg
dW5pdGVkMRMwEQYDVQQLDApTRUxGU0lHTkVEMQswCQYDVQQGEwJVUzAeFw0xODAx
MDQxNjE3MTRaFw0yODAxMDQxNjE3MTRaMGcxJjAkBgNVBAMMHWlQaG9uZSBEZXZl
bG9wZXI6IFNlbGYgU2lnbmVyMRswGQYDVQQKDBJTZWxmc2lnbmVycyB1bml0ZWQx
EzARBgNVBAsMClNFTEZTSUdORUQxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0B
AQEFAAOCAQ8AMIIBCgKCAQEAusfIxkCS6f9xQ/scxow4SIwP9ZLjicEP4FE962iN
1ywMt54wNvHu/VdeXaEOaOXZOwVyPDJ9N3L+f4X0V5MDFa47/R/YWWMCNEj5XxBe
SWZ8NB6CzcEaIxyRRyMtoKqIqa+HsPukw1XR0OJauR83rw/cdUuC+z6gT6jgAZvX
h/U2FM/F/fGOJq6wZySaj/+IuZ53SNE7tsn+uOdajXlEULDtWIAhtmt6IhIG7OxC
CaManYP8jqovgyu3cV65os5waBT3dhNKnhOLPP2q6dSa6qfqv/wAmmG0lgcJKZFE
OSNH4Y9eZB3TWQEuTX1uCLbPehpFZgQK0Z8p3ZsoehmDuQIDAQABozowODAOBgNV
HQ8BAf8EBAMCB4AwJgYDVR0lAQH/BBwwGgYIKwYBBQUHAwQGCCsGAQUFBwMDBgRV
HSUAMA0GCSqGSIb3DQEBCwUAA4IBAQCjm3J6GBKZia1YhGjMMmgTBXF50ZimQ/9n
MrFGgAq0WgrLgTJXuvYwvuw6tquo1foOvB8dYsn6W2II5zasUODWayixXdEB4kL1
buab4bNuJp5WXSFnDZz00GU9Mc4STLAFn1SZbOXlnFGwe1vSwUfqyAemHKqPh84v
3KOXIlPxRqguvNwhjJlaXJxv8MQHZnlTAjNqCsfQ4+zf8vieEzJJfBLGLq4bkXCT
hUipjmpvq19OuJju9tCDPc0RN5Si9dOkcFQw7c3/WAgNzDBk82lJcIuxAOa6M+Ds
waGeiXeezHAjvw60LyCpS9bB/5+vp+tPDhq2ByUP0YmfDbH6Jb+s
-----END CERTIFICATE-----
```

copy base64 value of certificate, join multilines into sing line and put it inside `<data>` tag:

```xml
<key>DeveloperCertificates</key>
<array>
    <data>MIIDgzCCAmugAwIBAgIBATANBgkqhkiG9w0BAQsFADBnMSYwJAYDVQQDDB1pUGhvbmUgRGV2ZWxvcGVyOiBTZWxmIFNpZ25lcjEbMBkGA1UECgwSU2VsZnNpZ25lcnMgdW5pdGVkMRMwEQYDVQQLDApTRUxGU0lHTkVEMQswCQYDVQQGEwJVUzAeFw0xODAxMDQxNjE3MTRaFw0yODAxMDQxNjE3MTRaMGcxJjAkBgNVBAMMHWlQaG9uZSBEZXZlbG9wZXI6IFNlbGYgU2lnbmVyMRswGQYDVQQKDBJTZWxmc2lnbmVycyB1bml0ZWQxEzARBgNVBAsMClNFTEZTSUdORUQxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAusfIxkCS6f9xQ/scxow4SIwP9ZLjicEP4FE962iN1ywMt54wNvHu/VdeXaEOaOXZOwVyPDJ9N3L+f4X0V5MDFa47/R/YWWMCNEj5XxBeSWZ8NB6CzcEaIxyRRyMtoKqIqa+HsPukw1XR0OJauR83rw/cdUuC+z6gT6jgAZvXh/U2FM/F/fGOJq6wZySaj/+IuZ53SNE7tsn+uOdajXlEULDtWIAhtmt6IhIG7OxCCaManYP8jqovgyu3cV65os5waBT3dhNKnhOLPP2q6dSa6qfqv/wAmmG0lgcJKZFEOSNH4Y9eZB3TWQEuTX1uCLbPehpFZgQK0Z8p3ZsoehmDuQIDAQABozowODAOBgNVHQ8BAf8EBAMCB4AwJgYDVR0lAQH/BBwwGgYIKwYBBQUHAwQGCCsGAQUFBwMDBgRVHSUAMA0GCSqGSIb3DQEBCwUAA4IBAQCjm3J6GBKZia1YhGjMMmgTBXF50ZimQ/9nMrFGgAq0WgrLgTJXuvYwvuw6tquo1foOvB8dYsn6W2II5zasUODWayixXdEB4kL1buab4bNuJp5WXSFnDZz00GU9Mc4STLAFn1SZbOXlnFGwe1vSwUfqyAemHKqPh84v3KOXIlPxRqguvNwhjJlaXJxv8MQHZnlTAjNqCsfQ4+zf8vieEzJJfBLGLq4bkXCThUipjmpvq19OuJju9tCDPc0RN5Si9dOkcFQw7c3/WAgNzDBk82lJcIuxAOa6M+DswaGeiXeezHAjvw60LyCpS9bB/5+vp+tPDhq2ByUP0YmfDbH6Jb+s</data>
</array>
```    

## final .plist looks as bellow
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>AppIDName</key>
	<string>selfsigned: any app</string>
	<key>ApplicationIdentifierPrefix</key>
	<array>
	<string>SELFSIGNED</string>
	</array>
	<key>CreationDate</key>
	<date>2018-01-01T13:00:00Z</date>
	<key>Platform</key>
	<array>
		<string>iOS</string>
	</array>
	<key>DeveloperCertificates</key>
    <array>
        <data>MIIDgzCCAmugAwIBAgIBATANBgkqhkiG9w0BAQsFADBnMSYwJAYDVQQDDB1pUGhvbmUgRGV2ZWxvcGVyOiBTZWxmIFNpZ25lcjEbMBkGA1UECgwSU2VsZnNpZ25lcnMgdW5pdGVkMRMwEQYDVQQLDApTRUxGU0lHTkVEMQswCQYDVQQGEwJVUzAeFw0xODAxMDQxNjE3MTRaFw0yODAxMDQxNjE3MTRaMGcxJjAkBgNVBAMMHWlQaG9uZSBEZXZlbG9wZXI6IFNlbGYgU2lnbmVyMRswGQYDVQQKDBJTZWxmc2lnbmVycyB1bml0ZWQxEzARBgNVBAsMClNFTEZTSUdORUQxCzAJBgNVBAYTAlVTMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAusfIxkCS6f9xQ/scxow4SIwP9ZLjicEP4FE962iN1ywMt54wNvHu/VdeXaEOaOXZOwVyPDJ9N3L+f4X0V5MDFa47/R/YWWMCNEj5XxBeSWZ8NB6CzcEaIxyRRyMtoKqIqa+HsPukw1XR0OJauR83rw/cdUuC+z6gT6jgAZvXh/U2FM/F/fGOJq6wZySaj/+IuZ53SNE7tsn+uOdajXlEULDtWIAhtmt6IhIG7OxCCaManYP8jqovgyu3cV65os5waBT3dhNKnhOLPP2q6dSa6qfqv/wAmmG0lgcJKZFEOSNH4Y9eZB3TWQEuTX1uCLbPehpFZgQK0Z8p3ZsoehmDuQIDAQABozowODAOBgNVHQ8BAf8EBAMCB4AwJgYDVR0lAQH/BBwwGgYIKwYBBQUHAwQGCCsGAQUFBwMDBgRVHSUAMA0GCSqGSIb3DQEBCwUAA4IBAQCjm3J6GBKZia1YhGjMMmgTBXF50ZimQ/9nMrFGgAq0WgrLgTJXuvYwvuw6tquo1foOvB8dYsn6W2II5zasUODWayixXdEB4kL1buab4bNuJp5WXSFnDZz00GU9Mc4STLAFn1SZbOXlnFGwe1vSwUfqyAemHKqPh84v3KOXIlPxRqguvNwhjJlaXJxv8MQHZnlTAjNqCsfQ4+zf8vieEzJJfBLGLq4bkXCThUipjmpvq19OuJju9tCDPc0RN5Si9dOkcFQw7c3/WAgNzDBk82lJcIuxAOa6M+DswaGeiXeezHAjvw60LyCpS9bB/5+vp+tPDhq2ByUP0YmfDbH6Jb+s</data>
    </array>
	<key>Entitlements</key>
	<dict>
		<key>keychain-access-groups</key>
		<array>
			<string>SELFSIGNED.*</string>
		</array>
		<key>get-task-allow</key>
		<false/>
		<key>application-identifier</key>
		<string>SELFSIGNED.*</string>
		<key>com.apple.developer.associated-domains</key>
		<string>*</string>
		<key>com.apple.developer.team-identifier</key>
		<string>SELFSIGNED</string>
		<key>aps-environment</key>
		<string>production</string>
	</dict>
	<key>ExpirationDate</key>
	<date>2028-01-01T14:00:00Z</date>
	<key>Name</key>
	<string>DO NOT USE: only for dummy signing</string>
	<key>ProvisionedDevices</key>
	<array>
	</array>
	<key>TeamIdentifier</key>
	<array>
		<string>SELFSIGNED</string>
	</array>
	<key>TeamName</key>
	<string>Selfsigners united</string>
	<key>TimeToLive</key>
	<integer>3652</integer>
	<key>UUID</key>
	<string>B5C2906D-D6EE-476E-AF17-D99AE14644AA</string>
	<key>Version</key>
	<integer>1</integer>
</dict>
</plist>
```

*optional* `TeamIdentifier`: replace all `SELFSIGNED` with value you have specified in `Organization Unit` of certificate
*optional* `UUID`: generate own one with `uuidgen`

## wrapping into CMS .mobileprovision
Turning .plist into .mobileprovision with CMS tool and using same certificate that was create for codesigning that is why it was created with `Any` and `Email Protection` options. Command line to sign:  
```
security cms -S -N "iOS Development: Self Signer" -i SelfSigned.plist -o SelfSigned.mobileprovision
```

Install resulted `SelfSigned.mobileprovision` as usual: click on it in Finder. If everything is set correctly and there is no mistake in dates it will be installed without any message.

## Testing
*Important:* Restart Xcode if it was running to allow it pick up created signing identity and installed provisioning profile

Once reopened Xcode will see provisioning profile but will not be able to pick certificate for it as team is unknown. Code signing shall be configured to manual mode:
* switch to `Build setting` tab and scroll down to `Signing` specification
* switch `Code sign style` to manual
* select created certificate `iPhone Developer: Self Signer` in `Code Signing Identity`
* select created profile `DO NOT USE: only for dummy signing` in `Provisioning Profile`
* keep team empty

Settings should looks similar to screenshot bellow:
 ![]({{ "/assets/2018/01/04/cert-using-in-xcode.png" | absolute_url}})   

Having these settings it is possible to build and dummy sign `Frameworks`, `plugins` or `application extensions`. Also it is possible to create and sign `.ipa` file this way but it is completely useless.
