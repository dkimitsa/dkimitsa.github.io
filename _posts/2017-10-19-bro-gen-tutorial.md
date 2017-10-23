---
layout: post
title: 'Tutorial: [ADMOB] using bro-gen to quick generate bindings'
tags: ['bro-gen', tutorial, binding]
---
It is not too big task to create java binding for custom framework as bro-gen semiautomatic tool available. This step-by-step tutorial explains how quickly generate binding using recently added suggestion feature. Admob is used here as an example framework.
<!-- more -->
***Dependencies***
Checkout
- Checkout [bro-gen](https://github.com/MobiVM/robovm-bro-gen)
- Checkout [RoboVM sources](https://github.com/MobiVM/robovm)

(please follow readme file for bro-gen to install dependencies)

***Important:*** make sure to put both bro-gen and robovm folders to same root with following names:
```
root
    robovm
    robovm-bro-gen
```

Create project folder structure similar to one used in robopods:
```
 <root>
    ios/src/main/bro-gen  (for yaml file )
    ios/src/main/java     (for generated bindings)
```

Download [admob 7.24.1](https://developers.google.com/admob/ios/download) and unpack to folder where yaml file will be created (e.g. ios/src/main/bro-gen)

***Create simple yaml file to start with***
To start empty yaml file that refers framework and list entity stubs to be created, save it as `googlemobileadssdk.yaml`
```yaml
package: org.robovm.pods.google
include: [foundation, uikit, storekit]
framework: GoogleMobileAds
clang_args: ['-x', 'objective-c']
headers:
    - GoogleMobileAds.h
typedefs: {}
enums: {}
classes: {}
protocols: {}

functions:
    # Make sure we don't miss any functions if new ones are introduced in a later version
    (GAD.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:
    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Constant__#{g[0]}'

```

Few comments about yaml:
- have specified package to put generated files: org.robovm.pods.google
- have listed system frameworks are used. to do this I just looked over all header files inside framework
- all groups (enums, classes, proto are left empty) as we will use suggestions from bro-gen to fill this up
- function/values/constants if any will go to FIXME class where we will look up for them
- *important*: do not use `library: GoogleMobileAds` as it means that framework is dynamicaly linked and this is not case of admob

*Round 0:* Starting script with following command line (current location is ios/src/main/bro-gen):
> ~/projects/git/robovm-bro-gen/bro-gen.rb ../java/ googlemobileadssdk.yaml

Parameters:
- ../java/ output folder where for generated files

***Important note about output folder.*** when generating bro-gen either updates existing file in output folder (e.g. uses it as template) or create new from template. Here is how template `templates/class_template.java` looks like:
```java
__LICENSE__
package org.robovm.foo;
/*<imports>*/
/*</imports>*/
/*<javadoc>*/
/*</javadoc>*/
/*<annotations>*/
/*</annotations>*/
/*<visibility>*/ public /*</visibility>*/ class /*<name>*/ TheName /*</name>*/
    extends /*<extends>*/ Object /*</extends>*/
    /*<implements>*//*</implements>*/ {

    /*<ptr>*/
    /*</ptr>*/
    /*<bind>*/
    /*</bind>*/
    /*<constants>*/
    /*</constants>*/
    /*<constructors>*/
    /*</constructors>*/
    /*<properties>*/
    /*</properties>*/
    /*<members>*/
    /*</members>*/
    /*<methods>*/
    /*</methods>*/
}
```

All generate code will be put between corresponding anchors. e.g. bro-gen will just do replace of `/*<constructors>*/ what ever /*</constructors>*/` with generated one. So if there is a need to add manually written code (like sweet sugar helpers or workaround) these to be put outside of anchors. It is important when updating iOS bindings as in cocoatouch there is a lot of hand written code.
```java
/*<visibility>*/ public /*</visibility>*/ class /*<name>*/ TheName /*</name>*/
    extends /*<extends>*/ Object /*</extends>*/
   // ...
    public TheName() {} // this constructor will survive bro-gen
    /*<constructors>*/

    // anything between anchors will be replaced during bro-gen run

    /*</constructors>*/
    // ...
}
```

*Back to script run.* It will produce bunch of warning messages that something is not known and lot of enums will go to FIXME class as constants (will handle this as next steps). What is useful it is suggestions output that list all data found in framework but missing in yaml. Console output of `bro-gen`:
```yaml
# YAML file pottentialy missing entries suggestions

#enums:
# potentialy missing enums
#enums:
    GADErrorCode: {prefix: kGADError}
    GADGender: {prefix: kGADGender}
    GADInAppPurchaseStatus: {prefix: kGADInAppPurchaseStatus}
    GADNativeAdImageAdLoaderOptionsOrientation: {}
    GADAdChoicesPosition: {}
    GADSearchBorderType: {prefix: kGADSearchBorderType}
    GADSearchCallButtonColor: {prefix: kGADSearchCallButton}
    GADMBannerAnimationType: {prefix: kGADMBannerAnimationType}

# potentialy missing structs
    GADAdSize: {}

# potentialy missing typedefs
    GADAdSize: {}

# classes to be updated:
classes:
    DFPBannerView: {}
    DFPBannerViewOptions: {}
    DFPCustomRenderedAd: {}
    DFPInterstitial: {}
    DFPRequest: {}
    GADAdChoicesView: {}
    GADAdLoader:
        methods:
            '-initWithAdUnitID:rootViewController:adTypes:options:':
                #trim_after_first_colon: true
                name: initWithAdUnitID$rootViewController$adTypes$options$
    GADAdLoaderOptions: {}
    GADAdReward:
        methods:
            '-initWithRewardType:rewardAmount:':
                #trim_after_first_colon: true
                name: initWithRewardType$rewardAmount$
    GADAudioVideoManager: {}
    GADBannerView:
        methods:
            '-initWithAdSize:origin:':
                #trim_after_first_colon: true
                name: initWithAdSize$origin$
    GADCorrelator: {}
    GADCorrelatorAdLoaderOptions: {}
    GADCustomEventExtras:
        methods:
            '-setExtras:forLabel:':
                #trim_after_first_colon: true
                name: setExtras$forLabel$
    GADCustomEventRequest: {}
    GADDebugOptionsViewController: {}
    GADDefaultInAppPurchase: {}
    GADDynamicHeightSearchRequest:
        methods:
            '-setAdvancedOptionValue:forKey:':
                #trim_after_first_colon: true
                name: setAdvancedOptionValue$forKey$
    GADExtras: {}
    GADInAppPurchase: {}
    GADInterstitial: {}
    GADMediaView: {}
    GADMediatedNativeAdNotificationSource: {}
    GADMobileAds:
        methods:
            '-isSDKVersionAtLeastMajor:minor:patch:':
                #trim_after_first_colon: true
                name: isSDKVersionAtLeastMajor$minor$patch$
    GADMultipleAdsAdLoaderOptions: {}
    GADNativeAd: {}
    GADNativeAdImage: {}
    GADNativeAdImageAdLoaderOptions: {}
    GADNativeAdViewAdOptions: {}
    GADNativeAppInstallAd:
        methods:
            '-registerAdView:assetViews:':
                #trim_after_first_colon: true
                name: registerAdView$assetViews$
    GADNativeAppInstallAdView: {}
    GADNativeContentAd:
        methods:
            '-registerAdView:assetViews:':
                #trim_after_first_colon: true
                name: registerAdView$assetViews$
    GADNativeContentAdView: {}
    GADNativeCustomTemplateAd:
        methods:
            '-performClickOnAssetWithKey:customClickHandler:':
                #trim_after_first_colon: true
                name: performClickOnAssetWithKey$customClickHandler$
    GADNativeExpressAdView:
        methods:
            '-initWithAdSize:origin:':
                #trim_after_first_colon: true
                name: initWithAdSize$origin$
    GADRequest:
        methods:
            '-setLocationWithLatitude:longitude:accuracy:':
                #trim_after_first_colon: true
                name: setLocationWithLatitude$longitude$accuracy$
            '-setBirthdayWithMonth:day:year:':
                #trim_after_first_colon: true
                name: setBirthdayWithMonth$day$year$
    GADRequestError: {}
    GADRewardBasedVideoAd:
        methods:
            '-loadRequest:withAdUnitID:':
                #trim_after_first_colon: true
                name: loadRequest$withAdUnitID$
    GADSearchBannerView: {}
    GADSearchRequest:
        methods:
            '-setBackgroundGradientFrom:toColor:':
                #trim_after_first_colon: true
                name: setBackgroundGradientFrom$toColor$
    GADVideoController: {}
    GADVideoOptions: {}



# protocols to be updated:
protocols:
    DFPBannerAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveDFPBannerView:':
                #trim_after_first_colon: true
                name: adLoader$didReceiveDFPBannerView$
    DFPCustomRenderedBannerViewDelegate:
        methods:
            '-bannerView:didReceiveCustomRenderedAd:':
                #trim_after_first_colon: true
                name: bannerView$didReceiveCustomRenderedAd$
    DFPCustomRenderedInterstitialDelegate:
        methods:
            '-interstitial:didReceiveCustomRenderedAd:':
                #trim_after_first_colon: true
                name: interstitial$didReceiveCustomRenderedAd$
    GADAdLoaderDelegate:
        methods:
            '-adLoader:didFailToReceiveAdWithError:':
                #trim_after_first_colon: true
                name: adLoader$didFailToReceiveAdWithError$
    GADAdNetworkExtras: {}
    GADAdSizeDelegate:
        methods:
            '-adView:willChangeAdSizeTo:':
                #trim_after_first_colon: true
                name: adView$willChangeAdSizeTo$
    GADAppEventDelegate:
        methods:
            '-adView:didReceiveAppEvent:withInfo:':
                #trim_after_first_colon: true
                name: adView$didReceiveAppEvent$withInfo$
            '-interstitial:didReceiveAppEvent:withInfo:':
                #trim_after_first_colon: true
                name: interstitial$didReceiveAppEvent$withInfo$
    GADAudioVideoManagerDelegate: {}
    GADBannerViewDelegate:
        methods:
            '-adView:didFailToReceiveAdWithError:':
                #trim_after_first_colon: true
                name: adView$didFailToReceiveAdWithError$
    GADCustomEventBanner:
        methods:
            '-requestBannerAd:parameter:label:request:':
                #trim_after_first_colon: true
                name: requestBannerAd$parameter$label$request$
    GADCustomEventBannerDelegate:
        methods:
            '-customEventBanner:didReceiveAd:':
                #trim_after_first_colon: true
                name: customEventBanner$didReceiveAd$
            '-customEventBanner:didFailAd:':
                #trim_after_first_colon: true
                name: customEventBanner$didFailAd$
            '-customEventBanner:clickDidOccurInAd:':
                #trim_after_first_colon: true
                name: customEventBanner$clickDidOccurInAd$
    GADCustomEventInterstitial:
        methods:
            '-requestInterstitialAdWithParameter:label:request:':
                #trim_after_first_colon: true
                name: requestInterstitialAdWithParameter$label$request$
    GADCustomEventInterstitialDelegate:
        methods:
            '-customEventInterstitial:didFailAd:':
                #trim_after_first_colon: true
                name: customEventInterstitial$didFailAd$
            '-customEventInterstitial:didReceiveAd:':
                #trim_after_first_colon: true
                name: customEventInterstitial$didReceiveAd$
    GADCustomEventNativeAd:
        methods:
            '-requestNativeAdWithParameter:request:adTypes:options:rootViewController:':
                #trim_after_first_colon: true
                name: requestNativeAdWithParameter$request$adTypes$options$rootViewController$
    GADCustomEventNativeAdDelegate:
        methods:
            '-customEventNativeAd:didReceiveMediatedNativeAd:':
                #trim_after_first_colon: true
                name: customEventNativeAd$didReceiveMediatedNativeAd$
            '-customEventNativeAd:didFailToLoadWithError:':
                #trim_after_first_colon: true
                name: customEventNativeAd$didFailToLoadWithError$
    GADDebugOptionsViewControllerDelegate: {}
    GADDefaultInAppPurchaseDelegate:
        methods:
            '-shouldStartPurchaseForProductID:quantity:':
                #trim_after_first_colon: true
                name: shouldStartPurchaseForProductID$quantity$
    GADInAppPurchaseDelegate: {}
    GADInterstitialDelegate:
        methods:
            '-interstitial:didFailToReceiveAdWithError:':
                #trim_after_first_colon: true
                name: interstitial$didFailToReceiveAdWithError$
    GADMAdNetworkAdapter:
        methods:
            '-getNativeAdWithAdTypes:options:':
                #trim_after_first_colon: true
                name: getNativeAdWithAdTypes$options$
    GADMAdNetworkConnector:
        methods:
            '-adapter:didFailAd:':
                #trim_after_first_colon: true
                name: adapter$didFailAd$
            '-adapter:didReceiveAdView:':
                #trim_after_first_colon: true
                name: adapter$didReceiveAdView$
            '-adapter:didReceiveMediatedNativeAd:':
                #trim_after_first_colon: true
                name: adapter$didReceiveMediatedNativeAd$
            '-adapter:didReceiveInterstitial:':
                #trim_after_first_colon: true
                name: adapter$didReceiveInterstitial$
            '-adapter:clickDidOccurInBanner:':
                #trim_after_first_colon: true
                name: adapter$clickDidOccurInBanner$
            '-adapter:didFailInterstitial:':
                #trim_after_first_colon: true
                name: adapter$didFailInterstitial$
    GADMRewardBasedVideoAdNetworkAdapter:
        methods:
            '-initWithRewardBasedVideoAdNetworkConnector:credentials:':
                #trim_after_first_colon: true
                name: initWithRewardBasedVideoAdNetworkConnector$credentials$
    GADMRewardBasedVideoAdNetworkConnector:
        methods:
            '-adapter:didFailToSetUpRewardBasedVideoAdWithError:':
                #trim_after_first_colon: true
                name: adapter$didFailToSetUpRewardBasedVideoAdWithError$
            '-adapter:didRewardUserWithReward:':
                #trim_after_first_colon: true
                name: adapter$didRewardUserWithReward$
            '-adapter:didFailToLoadRewardBasedVideoAdwithError:':
                #trim_after_first_colon: true
                name: adapter$didFailToLoadRewardBasedVideoAdwithError$
    GADMediatedNativeAd: {}
    GADMediatedNativeAdDelegate:
        methods:
            '-mediatedNativeAd:didRenderInView:viewController:':
                #trim_after_first_colon: true
                name: mediatedNativeAd$didRenderInView$viewController$
            '-mediatedNativeAd:didRecordClickOnAssetWithName:view:viewController:':
                #trim_after_first_colon: true
                name: mediatedNativeAd$didRecordClickOnAssetWithName$view$viewController$
            '-mediatedNativeAd:didUntrackView:':
                #trim_after_first_colon: true
                name: mediatedNativeAd$didUntrackView$
    GADMediatedNativeAppInstallAd: {}
    GADMediatedNativeContentAd: {}
    GADMediationAdRequest: {}
    GADNativeAdDelegate: {}
    GADNativeAppInstallAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeAppInstallAd:':
                #trim_after_first_colon: true
                name: adLoader$didReceiveNativeAppInstallAd$
    GADNativeContentAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeContentAd:':
                #trim_after_first_colon: true
                name: adLoader$didReceiveNativeContentAd$
    GADNativeCustomTemplateAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeCustomTemplateAd:':
                #trim_after_first_colon: true
                name: adLoader$didReceiveNativeCustomTemplateAd$
    GADNativeExpressAdViewDelegate:
        methods:
            '-nativeExpressAdView:didFailToReceiveAdWithError:':
                #trim_after_first_colon: true
                name: nativeExpressAdView$didFailToReceiveAdWithError$
    GADRewardBasedVideoAdDelegate:
        methods:
            '-rewardBasedVideoAd:didRewardUserWithReward:':
                #trim_after_first_colon: true
                name: rewardBasedVideoAd$didRewardUserWithReward$
            '-rewardBasedVideoAd:didFailToLoadWithError:':
                #trim_after_first_colon: true
                name: rewardBasedVideoAd$didFailToLoadWithError$
    GADVideoControllerDelegate: {}
```

in general suggestions are about:
- missing entities
- methods that have more than two arguments which will produce java names with $.

Just copy all these prepared entries to yaml, fixing method name on the go.
There is lot of stuff in FIXME so just lets ignore it for while.
One special moment about error codes that delivered through NSError code: enum GADErrorCode shall be marked as nserror one and domain variable shall go to it as well. More details in the [post]({{ site.baseurl }}{% post_url 2017-10-23-bro-gen-nserror-case %}). Modifications to `googlemobileadssdk.yaml`:

```yaml
enums:
    GADErrorCode: {nserror: true, prefix: kGADError}
....    
values:
    kGADErrorDomain:
        class: GADErrorCode
        name: getClassDomain
```

Also it is time to first look through FIXME class to pick up values. Just skip enums as these got there due unknown enums. Find similar looking values, *cross check* in headers and combine values to classes/enums. Modifications to `googlemobileadssdk.yaml`:
```yaml
values:
...
GADNativeContent(.*)Asset:
    class: GADNativeContentAdAssetIDs
    name: '#{g[0]}'

GADNativeAppInstall(.*)Asset:
    class: GADNativeAppInstallAdAssetIDs
    name: '#{g[0]}'

kGADAdSize(.*):
    class: GADAdSize
    name: '#{g[0]}'

kGADAdLoaderAdType(.*):
    class: GADAdLoaderAdTypes
    name: '#{g[0]}'
```

and functions:
```yaml
functions:
    GADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'
```
*no need to try cover everything in one yaml update, if miss anything just check FIXME again and make another round*

*Round 1*. Updated yaml is almost complete (most of stuff were just copied from suggestions). Now `googlemobileadssdk.yaml` contains:
```yaml
package: org.robovm.pods.google
include: [foundation, uikit, storekit]
framework: GoogleMobileAds
clang_args: ['-x', 'objective-c']
headers:
    - GoogleMobileAds.h
typedefs: {}

enums:
    GADErrorCode: {nserror: true, prefix: kGADError}
    GADGender: {prefix: kGADGender}
    GADInAppPurchaseStatus: {prefix: kGADInAppPurchaseStatus}
    GADNativeAdImageAdLoaderOptionsOrientation: {}
    GADAdChoicesPosition: {}
    GADSearchBorderType: {prefix: kGADSearchBorderType}
    GADSearchCallButtonColor: {prefix: kGADSearchCallButton}
    GADMBannerAnimationType: {prefix: kGADMBannerAnimationType}

classes:
    GADAdSize: {} # struct
    DFPBannerView: {}
    DFPBannerViewOptions: {}
    DFPCustomRenderedAd: {}
    DFPInterstitial: {}
    DFPRequest: {}
    GADAdChoicesView: {}
    GADAdLoader:
        methods:
            '-initWithAdUnitID:rootViewController:adTypes:options:':
                name: init
    GADAdLoaderOptions: {}
    GADAdReward:
        methods:
            '-initWithRewardType:rewardAmount:':
                name: init
    GADAudioVideoManager: {}
    GADBannerView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADCorrelator: {}
    GADCorrelatorAdLoaderOptions: {}
    GADCustomEventExtras:
        methods:
            '-setExtras:forLabel:':
                trim_after_first_colon: true
    GADCustomEventRequest: {}
    GADDebugOptionsViewController: {}
    GADDefaultInAppPurchase: {}
    GADDynamicHeightSearchRequest:
        methods:
            '-setAdvancedOptionValue:forKey:':
                #trim_after_first_colon: true
                name: setAdvancedOptionValue$forKey$
    GADExtras: {}
    GADInAppPurchase: {}
    GADInterstitial: {}
    GADMediaView: {}
    GADMediatedNativeAdNotificationSource: {}
    GADMobileAds:
        methods:
            '-isSDKVersionAtLeastMajor:minor:patch:':
                name: isSDKVersionAtLeast
    GADMultipleAdsAdLoaderOptions: {}
    GADNativeAd: {}
    GADNativeAdImage: {}
    GADNativeAdImageAdLoaderOptions: {}
    GADNativeAdViewAdOptions: {}
    GADNativeAppInstallAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeAppInstallAdView: {}
    GADNativeContentAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeContentAdView: {}
    GADNativeCustomTemplateAd:
        methods:
            '-performClickOnAssetWithKey:customClickHandler:':
                name: performClickOnAsset
    GADNativeExpressAdView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADRequest:
        methods:
            '-setLocationWithLatitude:longitude:accuracy:':
                name: setLocation
            '-setBirthdayWithMonth:day:year:':
                name: setBirthday
    GADRequestError: {}
    GADRewardBasedVideoAd:
        methods:
            '-loadRequest:withAdUnitID:':
                trim_after_first_colon: true
    GADSearchBannerView: {}
    GADSearchRequest:
        methods:
            '-setBackgroundGradientFrom:toColor:':
                name: setBackgroundGradient
    GADVideoController: {}
    GADVideoOptions: {}


protocols:
    DFPBannerAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveDFPBannerView:':
                name: didReceiveDFPBannerView
    DFPCustomRenderedBannerViewDelegate:
        methods:
            '-bannerView:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    DFPCustomRenderedInterstitialDelegate:
        methods:
            '-interstitial:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    GADAdLoaderDelegate:
        methods:
            '-adLoader:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADAdNetworkExtras: {}
    GADAdSizeDelegate:
        methods:
            '-adView:willChangeAdSizeTo:':
                name: willChangeAdSize
    GADAppEventDelegate:
        methods:
            '-adView:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
            '-interstitial:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
    GADAudioVideoManagerDelegate: {}
    GADBannerViewDelegate:
        methods:
            '-adView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADCustomEventBanner:
        methods:
            '-requestBannerAd:parameter:label:request:':
                name: requestBanner
    GADCustomEventBannerDelegate:
        methods:
            '-customEventBanner:didReceiveAd:':
                name: didReceiveAd
            '-customEventBanner:didFailAd:':
                name: didFailAd
            '-customEventBanner:clickDidOccurInAd:':
                name: clickDidOccurInAd
    GADCustomEventInterstitial:
        methods:
            '-requestInterstitialAdWithParameter:label:request:':
                name: requestInterstitialAd
    GADCustomEventInterstitialDelegate:
        methods:
            '-customEventInterstitial:didFailAd:':
                name: didFailAd
            '-customEventInterstitial:didReceiveAd:':
                name: didReceiveAd
    GADCustomEventNativeAd:
        methods:
            '-requestNativeAdWithParameter:request:adTypes:options:rootViewController:':
                name: requestNativeAdW
    GADCustomEventNativeAdDelegate:
        methods:
            '-customEventNativeAd:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-customEventNativeAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADDebugOptionsViewControllerDelegate: {}
    GADDefaultInAppPurchaseDelegate:
        methods:
            '-shouldStartPurchaseForProductID:quantity:':
                name: shouldStartPurchase
    GADInAppPurchaseDelegate: {}
    GADInterstitialDelegate:
        methods:
            '-interstitial:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADMAdNetworkAdapter:
        methods:
            '-getNativeAdWithAdTypes:options:':
                name: getNativeAd
    GADMAdNetworkConnector:
        methods:
            '-adapter:didFailAd:':
                name: didFailAd
            '-adapter:didReceiveAdView:':
                name: didReceiveAdView
            '-adapter:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-adapter:didReceiveInterstitial:':
                name: didReceiveInterstitial
            '-adapter:clickDidOccurInBanner:':
                name: clickDidOccurInBanner
            '-adapter:didFailInterstitial:':
                name: didFailInterstitial
    GADMRewardBasedVideoAdNetworkAdapter:
        methods:
            '-initWithRewardBasedVideoAdNetworkConnector:credentials:':
                name: createAdapter
    GADMRewardBasedVideoAdNetworkConnector:
        methods:
            '-adapter:didFailToSetUpRewardBasedVideoAdWithError:':
                name: didFailToSetUpRewardBasedVideoAd
            '-adapter:didRewardUserWithReward:':
                name: didRewardUserWithReward
            '-adapter:didFailToLoadRewardBasedVideoAdwithError:':
                name: didFailToLoadRewardBasedVideoAd
    GADMediatedNativeAd: {}
    GADMediatedNativeAdDelegate:
        methods:
            '-mediatedNativeAd:didRenderInView:viewController:':
                name: didRenderInView
            '-mediatedNativeAd:didRecordClickOnAssetWithName:view:viewController:':
                name: didRecordClickOnAsset
            '-mediatedNativeAd:didUntrackView:':
                name: didUntrackView
    GADMediatedNativeAppInstallAd: {}
    GADMediatedNativeContentAd: {}
    GADMediationAdRequest: {}
    GADNativeAdDelegate: {}
    GADNativeAppInstallAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeAppInstallAd:':
                name: didReceiveNativeAppInstallAd
    GADNativeContentAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeContentAd:':
                name: didReceiveNativeContentAd
    GADNativeCustomTemplateAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeCustomTemplateAd:':
                name: didReceiveNativeCustomTemplateAd
    GADNativeExpressAdViewDelegate:
        methods:
            '-nativeExpressAdView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADRewardBasedVideoAdDelegate:
        methods:
            '-rewardBasedVideoAd:didRewardUserWithReward:':
                name: didRewardUser
            '-rewardBasedVideoAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADVideoControllerDelegate: {}

functions:
    GADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    # Make sure we don't miss any functions if new ones are introduced in a later version
    (GAD.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:
    kGADErrorDomain:
        class: GADErrorCode
        name: getClassDomain

    GADNativeContent(.*)Asset:
        class: GADNativeContentAdAssetIDs
        name: '#{g[0]}'

    GADNativeAppInstall(.*)Asset:
        class: GADNativeAppInstallAdAssetIDs
        name: '#{g[0]}'

    kGADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    kGADAdLoaderAdType(.*):
        class: GADAdLoaderAdTypes
        name: '#{g[0]}'

    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Constant__#{g[0]}'
```

Suggestion list is tiny this time (just one initializer that was not picked by bro-gen first run). Console output of `bro-gen`:
```yaml
# classes to be updated:
classes:
    GADNativeAdImage:
        methods:
            '-initWithURL:scale:':
                #trim_after_first_colon: true
                name: initWithURL$scale$
```

And `FIXME.java` contains less of entries as well:
```java
@GlobalValue(symbol="kGADSimulatorID", optional=true)
public static native NSObject Value__GADSimulatorID();
@GlobalValue(symbol="GADNativeCustomTemplateAdMediaViewKey", optional=true)
public static native String Value__GADNativeCustomTemplateAdMediaViewKey();
@GlobalValue(symbol="GADCustomEventParametersServer", optional=true)
public static native String Value__GADCustomEventParametersServer();
```

Fixing values in `googlemobileadssdk.yaml`:
```yaml
values:
...
    kGADSimulatorID:
        class: GADRequest
        name: getSimulatorID

    GADNativeCustomTemplateAdMediaViewKey:
        class: GADNativeCustomTemplateAd
        name: getCustomTemplateAdMediaViewKey

    GADCustomEventParameters(Server):
        class: GADCustomEventParameters
        name: '#{g[0]}'
```

*Round 2*. Regenerating. Content of `googlemobileadssdk.yaml` now as bellow:
```yaml
package: org.robovm.pods.google
include: [foundation, uikit, storekit]
framework: GoogleMobileAds
clang_args: ['-x', 'objective-c']
headers:
    - GoogleMobileAds.h
typedefs: {}

enums:
    GADErrorCode: {nserror: true, prefix: kGADError}
    GADGender: {prefix: kGADGender}
    GADInAppPurchaseStatus: {prefix: kGADInAppPurchaseStatus}
    GADNativeAdImageAdLoaderOptionsOrientation: {}
    GADAdChoicesPosition: {}
    GADSearchBorderType: {prefix: kGADSearchBorderType}
    GADSearchCallButtonColor: {prefix: kGADSearchCallButton}
    GADMBannerAnimationType: {prefix: kGADMBannerAnimationType}

classes:
    GADAdSize: {} # struct
    DFPBannerView: {}
    DFPBannerViewOptions: {}
    DFPCustomRenderedAd: {}
    DFPInterstitial: {}
    DFPRequest: {}
    GADAdChoicesView: {}
    GADAdLoader:
        methods:
            '-initWithAdUnitID:rootViewController:adTypes:options:':
                name: init
    GADAdLoaderOptions: {}
    GADAdReward:
        methods:
            '-initWithRewardType:rewardAmount:':
                name: init
    GADAudioVideoManager: {}
    GADBannerView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADCorrelator: {}
    GADCorrelatorAdLoaderOptions: {}
    GADCustomEventExtras:
        methods:
            '-setExtras:forLabel:':
                trim_after_first_colon: true
    GADCustomEventRequest: {}
    GADDebugOptionsViewController: {}
    GADDefaultInAppPurchase: {}
    GADDynamicHeightSearchRequest:
        methods:
            '-setAdvancedOptionValue:forKey:':
                #trim_after_first_colon: true
                name: setAdvancedOptionValue$forKey$
    GADExtras: {}
    GADInAppPurchase: {}
    GADInterstitial: {}
    GADMediaView: {}
    GADMediatedNativeAdNotificationSource: {}
    GADMobileAds:
        methods:
            '-isSDKVersionAtLeastMajor:minor:patch:':
                name: isSDKVersionAtLeast
    GADMultipleAdsAdLoaderOptions: {}
    GADNativeAd: {}
    GADNativeAdImage:
        methods:
            '-initWithURL:scale:':
                name: init
    GADNativeAdImageAdLoaderOptions: {}
    GADNativeAdViewAdOptions: {}
    GADNativeAppInstallAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeAppInstallAdView: {}
    GADNativeContentAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeContentAdView: {}
    GADNativeCustomTemplateAd:
        methods:
            '-performClickOnAssetWithKey:customClickHandler:':
                name: performClickOnAsset
    GADNativeExpressAdView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADRequest:
        methods:
            '-setLocationWithLatitude:longitude:accuracy:':
                name: setLocation
            '-setBirthdayWithMonth:day:year:':
                name: setBirthday
    GADRequestError: {}
    GADRewardBasedVideoAd:
        methods:
            '-loadRequest:withAdUnitID:':
                trim_after_first_colon: true
    GADSearchBannerView: {}
    GADSearchRequest:
        methods:
            '-setBackgroundGradientFrom:toColor:':
                name: setBackgroundGradient
    GADVideoController: {}
    GADVideoOptions: {}


protocols:
    DFPBannerAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveDFPBannerView:':
                name: didReceiveDFPBannerView
    DFPCustomRenderedBannerViewDelegate:
        methods:
            '-bannerView:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    DFPCustomRenderedInterstitialDelegate:
        methods:
            '-interstitial:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    GADAdLoaderDelegate:
        methods:
            '-adLoader:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADAdNetworkExtras: {}
    GADAdSizeDelegate:
        methods:
            '-adView:willChangeAdSizeTo:':
                name: willChangeAdSize
    GADAppEventDelegate:
        methods:
            '-adView:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
            '-interstitial:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
    GADAudioVideoManagerDelegate: {}
    GADBannerViewDelegate:
        methods:
            '-adView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADCustomEventBanner:
        methods:
            '-requestBannerAd:parameter:label:request:':
                name: requestBanner
    GADCustomEventBannerDelegate:
        methods:
            '-customEventBanner:didReceiveAd:':
                name: didReceiveAd
            '-customEventBanner:didFailAd:':
                name: didFailAd
            '-customEventBanner:clickDidOccurInAd:':
                name: clickDidOccurInAd
    GADCustomEventInterstitial:
        methods:
            '-requestInterstitialAdWithParameter:label:request:':
                name: requestInterstitialAd
    GADCustomEventInterstitialDelegate:
        methods:
            '-customEventInterstitial:didFailAd:':
                name: didFailAd
            '-customEventInterstitial:didReceiveAd:':
                name: didReceiveAd
    GADCustomEventNativeAd:
        methods:
            '-requestNativeAdWithParameter:request:adTypes:options:rootViewController:':
                name: requestNativeAdW
    GADCustomEventNativeAdDelegate:
        methods:
            '-customEventNativeAd:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-customEventNativeAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADDebugOptionsViewControllerDelegate: {}
    GADDefaultInAppPurchaseDelegate:
        methods:
            '-shouldStartPurchaseForProductID:quantity:':
                name: shouldStartPurchase
    GADInAppPurchaseDelegate: {}
    GADInterstitialDelegate:
        methods:
            '-interstitial:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADMAdNetworkAdapter:
        methods:
            '-getNativeAdWithAdTypes:options:':
                name: getNativeAd
    GADMAdNetworkConnector:
        methods:
            '-adapter:didFailAd:':
                name: didFailAd
            '-adapter:didReceiveAdView:':
                name: didReceiveAdView
            '-adapter:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-adapter:didReceiveInterstitial:':
                name: didReceiveInterstitial
            '-adapter:clickDidOccurInBanner:':
                name: clickDidOccurInBanner
            '-adapter:didFailInterstitial:':
                name: didFailInterstitial
    GADMRewardBasedVideoAdNetworkAdapter:
        methods:
            '-initWithRewardBasedVideoAdNetworkConnector:credentials:':
                name: createAdapter
    GADMRewardBasedVideoAdNetworkConnector:
        methods:
            '-adapter:didFailToSetUpRewardBasedVideoAdWithError:':
                name: didFailToSetUpRewardBasedVideoAd
            '-adapter:didRewardUserWithReward:':
                name: didRewardUserWithReward
            '-adapter:didFailToLoadRewardBasedVideoAdwithError:':
                name: didFailToLoadRewardBasedVideoAd
    GADMediatedNativeAd: {}
    GADMediatedNativeAdDelegate:
        methods:
            '-mediatedNativeAd:didRenderInView:viewController:':
                name: didRenderInView
            '-mediatedNativeAd:didRecordClickOnAssetWithName:view:viewController:':
                name: didRecordClickOnAsset
            '-mediatedNativeAd:didUntrackView:':
                name: didUntrackView
    GADMediatedNativeAppInstallAd: {}
    GADMediatedNativeContentAd: {}
    GADMediationAdRequest: {}
    GADNativeAdDelegate: {}
    GADNativeAppInstallAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeAppInstallAd:':
                name: didReceiveNativeAppInstallAd
    GADNativeContentAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeContentAd:':
                name: didReceiveNativeContentAd
    GADNativeCustomTemplateAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeCustomTemplateAd:':
                name: didReceiveNativeCustomTemplateAd
    GADNativeExpressAdViewDelegate:
        methods:
            '-nativeExpressAdView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADRewardBasedVideoAdDelegate:
        methods:
            '-rewardBasedVideoAd:didRewardUserWithReward:':
                name: didRewardUser
            '-rewardBasedVideoAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADVideoControllerDelegate: {}

functions:
    GADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    # Make sure we don't miss any functions if new ones are introduced in a later version
    (GAD.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:
    kGADErrorDomain:
        class: GADErrorCode
        name: getClassDomain

    GADNativeContent(.*)Asset:
        class: GADNativeContentAdAssetIDs
        name: '#{g[0]}'

    GADNativeAppInstall(.*)Asset:
        class: GADNativeAppInstallAdAssetIDs
        name: '#{g[0]}'

    kGADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    kGADAdLoaderAdType(.*):
        class: GADAdLoaderAdTypes
        name: '#{g[0]}'

    kGADSimulatorID:
        class: GADRequest
        name: getSimulatorID

    GADNativeCustomTemplateAdMediaViewKey:
        class: GADNativeCustomTemplateAd
        name: getCustomTemplateAdMediaViewKey

    GADCustomEventParameters(Server):
        class: GADCustomEventParameters
        name: '#{g[0]}'

    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Constant__#{g[0]}'
```

Looks ok, *trying to compile*. Bunch of errors:
1. cannot find symbol GADAdLoaderAdType, it is typedef to NSString in objc source, fixing this with private typedef in `googlemobileadssdk.yaml`
```
private_typedefs:
    GADAdLoaderAdType: NSString
...    
values:
...
    kGADAdLoaderAdType(.*):
        class: GADAdLoaderAdTypes
        type: NSString
        name: '#{g[0]}'
```

2. cannot find symbol CGPoint -- need to refer coregraphics in `googlemobileadssdk.yaml`
>include: [foundation, uikit, storekit, coregraphics]

3. GADRequestError: Error:(47, 30) java: no suitable constructor found for NSError(no arguments).
Google adds own subclass for NSError, ok, lets just remove default constructor it complaints about (as user will not create this kind of object by itself). Update to `googlemobileadssdk.yaml`:
```yaml
classes:
    GADRequestError: {skip_def_constructor: true}
```

Final `googlemobileadssdk.yaml`:
```yaml
package: org.robovm.pods.google
include: [foundation, uikit, storekit, coregraphics]
framework: GoogleMobileAds
clang_args: ['-x', 'objective-c']
headers:
    - GoogleMobileAds.h

private_typedefs:
    GADAdLoaderAdType: NSString

typedefs: {}

enums:
    GADErrorCode: {nserror: true, prefix: kGADError}
    GADGender: {prefix: kGADGender}
    GADInAppPurchaseStatus: {prefix: kGADInAppPurchaseStatus}
    GADNativeAdImageAdLoaderOptionsOrientation: {}
    GADAdChoicesPosition: {}
    GADSearchBorderType: {prefix: kGADSearchBorderType}
    GADSearchCallButtonColor: {prefix: kGADSearchCallButton}
    GADMBannerAnimationType: {prefix: kGADMBannerAnimationType}

classes:
    GADAdSize: {} # struct
    DFPBannerView: {}
    DFPBannerViewOptions: {}
    DFPCustomRenderedAd: {}
    DFPInterstitial: {}
    DFPRequest: {}
    GADAdChoicesView: {}
    GADAdLoader:
        methods:
            '-initWithAdUnitID:rootViewController:adTypes:options:':
                name: init
    GADAdLoaderOptions: {}
    GADAdReward:
        methods:
            '-initWithRewardType:rewardAmount:':
                name: init
    GADAudioVideoManager: {}
    GADBannerView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADCorrelator: {}
    GADCorrelatorAdLoaderOptions: {}
    GADCustomEventExtras:
        methods:
            '-setExtras:forLabel:':
                trim_after_first_colon: true
    GADCustomEventRequest: {}
    GADDebugOptionsViewController: {}
    GADDefaultInAppPurchase: {}
    GADDynamicHeightSearchRequest:
        methods:
            '-setAdvancedOptionValue:forKey:':
                #trim_after_first_colon: true
                name: setAdvancedOptionValue$forKey$
    GADExtras: {}
    GADInAppPurchase: {}
    GADInterstitial: {}
    GADMediaView: {}
    GADMediatedNativeAdNotificationSource: {}
    GADMobileAds:
        methods:
            '-isSDKVersionAtLeastMajor:minor:patch:':
                name: isSDKVersionAtLeast
    GADMultipleAdsAdLoaderOptions: {}
    GADNativeAd: {}
    GADNativeAdImage:
        methods:
            '-initWithURL:scale:':
                name: init
    GADNativeAdImageAdLoaderOptions: {}
    GADNativeAdViewAdOptions: {}
    GADNativeAppInstallAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeAppInstallAdView: {}
    GADNativeContentAd:
        methods:
            '-registerAdView:assetViews:':
                trim_after_first_colon: true
    GADNativeContentAdView: {}
    GADNativeCustomTemplateAd:
        methods:
            '-performClickOnAssetWithKey:customClickHandler:':
                name: performClickOnAsset
    GADNativeExpressAdView:
        methods:
            '-initWithAdSize:origin:':
                name: init
    GADRequest:
        methods:
            '-setLocationWithLatitude:longitude:accuracy:':
                name: setLocation
            '-setBirthdayWithMonth:day:year:':
                name: setBirthday
    GADRequestError: {skip_def_constructor: true}
    GADRewardBasedVideoAd:
        methods:
            '-loadRequest:withAdUnitID:':
                trim_after_first_colon: true
    GADSearchBannerView: {}
    GADSearchRequest:
        methods:
            '-setBackgroundGradientFrom:toColor:':
                name: setBackgroundGradient
    GADVideoController: {}
    GADVideoOptions: {}


protocols:
    DFPBannerAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveDFPBannerView:':
                name: didReceiveDFPBannerView
    DFPCustomRenderedBannerViewDelegate:
        methods:
            '-bannerView:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    DFPCustomRenderedInterstitialDelegate:
        methods:
            '-interstitial:didReceiveCustomRenderedAd:':
                name: didReceiveCustomRenderedAd
    GADAdLoaderDelegate:
        methods:
            '-adLoader:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADAdNetworkExtras: {}
    GADAdSizeDelegate:
        methods:
            '-adView:willChangeAdSizeTo:':
                name: willChangeAdSize
    GADAppEventDelegate:
        methods:
            '-adView:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
            '-interstitial:didReceiveAppEvent:withInfo:':
                name: didReceiveAppEvent
    GADAudioVideoManagerDelegate: {}
    GADBannerViewDelegate:
        methods:
            '-adView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADCustomEventBanner:
        methods:
            '-requestBannerAd:parameter:label:request:':
                name: requestBanner
    GADCustomEventBannerDelegate:
        methods:
            '-customEventBanner:didReceiveAd:':
                name: didReceiveAd
            '-customEventBanner:didFailAd:':
                name: didFailAd
            '-customEventBanner:clickDidOccurInAd:':
                name: clickDidOccurInAd
    GADCustomEventInterstitial:
        methods:
            '-requestInterstitialAdWithParameter:label:request:':
                name: requestInterstitialAd
    GADCustomEventInterstitialDelegate:
        methods:
            '-customEventInterstitial:didFailAd:':
                name: didFailAd
            '-customEventInterstitial:didReceiveAd:':
                name: didReceiveAd
    GADCustomEventNativeAd:
        methods:
            '-requestNativeAdWithParameter:request:adTypes:options:rootViewController:':
                name: requestNativeAdW
    GADCustomEventNativeAdDelegate:
        methods:
            '-customEventNativeAd:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-customEventNativeAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADDebugOptionsViewControllerDelegate: {}
    GADDefaultInAppPurchaseDelegate:
        methods:
            '-shouldStartPurchaseForProductID:quantity:':
                name: shouldStartPurchase
    GADInAppPurchaseDelegate: {}
    GADInterstitialDelegate:
        methods:
            '-interstitial:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADMAdNetworkAdapter:
        methods:
            '-getNativeAdWithAdTypes:options:':
                name: getNativeAd
    GADMAdNetworkConnector:
        methods:
            '-adapter:didFailAd:':
                name: didFailAd
            '-adapter:didReceiveAdView:':
                name: didReceiveAdView
            '-adapter:didReceiveMediatedNativeAd:':
                name: didReceiveMediatedNativeAd
            '-adapter:didReceiveInterstitial:':
                name: didReceiveInterstitial
            '-adapter:clickDidOccurInBanner:':
                name: clickDidOccurInBanner
            '-adapter:didFailInterstitial:':
                name: didFailInterstitial
    GADMRewardBasedVideoAdNetworkAdapter:
        methods:
            '-initWithRewardBasedVideoAdNetworkConnector:credentials:':
                name: createAdapter
    GADMRewardBasedVideoAdNetworkConnector:
        methods:
            '-adapter:didFailToSetUpRewardBasedVideoAdWithError:':
                name: didFailToSetUpRewardBasedVideoAd
            '-adapter:didRewardUserWithReward:':
                name: didRewardUserWithReward
            '-adapter:didFailToLoadRewardBasedVideoAdwithError:':
                name: didFailToLoadRewardBasedVideoAd
    GADMediatedNativeAd: {}
    GADMediatedNativeAdDelegate:
        methods:
            '-mediatedNativeAd:didRenderInView:viewController:':
                name: didRenderInView
            '-mediatedNativeAd:didRecordClickOnAssetWithName:view:viewController:':
                name: didRecordClickOnAsset
            '-mediatedNativeAd:didUntrackView:':
                name: didUntrackView
    GADMediatedNativeAppInstallAd: {}
    GADMediatedNativeContentAd: {}
    GADMediationAdRequest: {}
    GADNativeAdDelegate: {}
    GADNativeAppInstallAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeAppInstallAd:':
                name: didReceiveNativeAppInstallAd
    GADNativeContentAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeContentAd:':
                name: didReceiveNativeContentAd
    GADNativeCustomTemplateAdLoaderDelegate:
        methods:
            '-adLoader:didReceiveNativeCustomTemplateAd:':
                name: didReceiveNativeCustomTemplateAd
    GADNativeExpressAdViewDelegate:
        methods:
            '-nativeExpressAdView:didFailToReceiveAdWithError:':
                name: didFailToReceiveAd
    GADRewardBasedVideoAdDelegate:
        methods:
            '-rewardBasedVideoAd:didRewardUserWithReward:':
                name: didRewardUser
            '-rewardBasedVideoAd:didFailToLoadWithError:':
                name: didFailToLoad
    GADVideoControllerDelegate: {}

functions:
    GADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    # Make sure we don't miss any functions if new ones are introduced in a later version
    (GAD.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:
    kGADErrorDomain:
        class: GADErrorCode
        name: getClassDomain

    GADNativeContent(.*)Asset:
        class: GADNativeContentAdAssetIDs
        name: '#{g[0]}'

    GADNativeAppInstall(.*)Asset:
        class: GADNativeAppInstallAdAssetIDs
        name: '#{g[0]}'

    kGADAdSize(.*):
        class: GADAdSize
        name: '#{g[0]}'

    kGADAdLoaderAdType(.*):
        class: GADAdLoaderAdTypes
        type: NSString
        name: '#{g[0]}'

    kGADSimulatorID:
        class: GADRequest
        name: getSimulatorID

    GADNativeCustomTemplateAdMediaViewKey:
        class: GADNativeCustomTemplateAd
        name: getCustomTemplateAdMediaViewKey

    GADCustomEventParameters(Server):
        class: GADCustomEventParameters
        name: '#{g[0]}'

    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(GAD.*):
        class: FIXME
        name: 'Constant__#{g[0]}'
```

*Round 3*. Regenerating. *IMPORTANT* don't forget to delete FIXME.java and it shall not be generated again.  

***Bindings are ready. Verification time.***  
Total: 125 files and 300K of code was generated  
Will not create robopod for this instead will just copy bindings into project.  
Quick porting [RewardedVideoExample](https://github.com/googleads/googleads-mobile-ios-examples/tree/master/Objective-C/admob/RewardedVideoExample)  
And RoboVM project at [github](https://github.com/dkimitsa/codesnippets/tree/master/RewardedVideoExample). (you have to put framework to ext/ folder)
