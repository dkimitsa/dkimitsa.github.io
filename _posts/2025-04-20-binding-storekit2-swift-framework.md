---
layout: post
title: 'StoreKit2: bindings for pure swift framework'
tags: [binding, swift, tutorial]
---
It was requested for a while as StoreKit is marked as deprecated.

Moments with it is that swift doesn't provide runtime to work with its classes from other languages as result Java classes can't be bound to native pure switch similar as bro-bridge does.
<!-- more -->
Same thing affects other tools:  
[Kotlin](https://kotlinlang.org/docs/native-objc-interop.html#usage):
> A Swift library can be used in Kotlin code if its API is exported to Objective-C with @objc. Pure Swift modules are not yet supported.

[Xamarin](https://learn.microsoft.com/en-us/previous-versions/xamarin/ios/platform/binding-swift/walkthrough)
> If the header doesn't exist or has an incomplete public interface (for example, you don't see classes/members) you have two options:
> -Update the Swift source code to generate the header and mark the required members with @objc attribute
> - Build a proxy framework where you control the public interface and proxy all the calls to underlying framework


Solution is to wrap pure swift code with objective c compatible code (Kotlin native and Xamarin suggest same) and then bind it using bro-gen into java.
As StoreKit2 is concurrent api it good idea to wrap it into Kotlin suspendable API otherwise it will be callback oriented code.

All ObjC wrapper, Java bindings and Kotlin extensions were pushed into https://github.com/dkimitsa/robovm-cocoatouch-swift/tree/main/storekit repository.

Artefact was pushed to sonatype snapshot repository and available under following ids:

```groovy
repositories {
    maven { url 'https://oss.sonatype.org/content/repositories/snapshots' }
}
dependencies {
    implementation "io.github.dkimitsa.robovm:robopods-storekit-swift:18.2.0.1-SNAPSHOT"
}

```

Usage is similar as per Apple docs:   
Java:
```Java
Product.getProducts(NSArray.fromStrings("product.id.1"), (products, nsError) -> {
    if (nsError != null) {
        // handle error
        return;
    }
    // handle products
});
```
or kotlin:
```kotlin
scope.launch(Dispatchers.Main) {
    try {
        val storeProducts = ProductKt.getProducts(listOf("product.id.1"))
    } catch(e: NSErrorException) {
        e.printStackTrace()
    }
}
```

## Kotlin
Kotlin extensions were added to Java classes that turns most of the functions with callbacks into suspendable. This allows to use API as designed by Apple as StoreKit2 is written with idea of structured concurrency.
As kotlin can't add `Companion` to Java classes to extend these with static methods, following objects were introduced and extensions added to them:
```kotlin
object AppStoreKt
object ProductKt {
    object PromotionInfoKt
    object SubscriptionInfoKt
}
object TransactionKt
object AppTransactionKt
object ExternalLinkAccountKt
object ExternalPurchaseKt
object ExternalPurchaseCustomLinkKt
object ExternalPurchaseLinkKt
object PaymentMethodBindingKt
object StorefrontKt
```

example:
```kotlin
/**
 * @since Available in iOS 16.0 and later.
 * @throws org.robovm.apple.foundation.NSErrorException
 */
suspend fun AppStoreKt.presentOfferCodeRedeemSheet(scene: UIWindowScene) = suspendCancellableTask { cont ->
    AppStore.presentOfferCodeRedeemSheet(scene, cont::completionHandler)
}
```

WARNING: suspendable functions are not returning NSError object but will throw corresponding exception, make sure to handle these.


## Sample app
Sample Apple's Implementing a store in your app using the StoreKit API was ported into Kotlin as proof of concept and available as part of alt-pods-tests repository:
https://github.com/dkimitsa/alt-pods-tests/tree/master/cocoatouch-swift-storekit .


Probably there are bugs or some api might be missing. Please open an [issue](https://github.com/dkimitsa/robovm-cocoatouch-swift/issues).
