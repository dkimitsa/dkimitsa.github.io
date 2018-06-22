---
layout: post
title: 'Why RoboVM crashes in Antarctica'
tags: [fix, lol, 'icu.dat']
---

Spoiler: because Apple uses Penguins to review application in Antarctica! (or just set region to Antarctica)  

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

`Currency.getInstance()` can return null [as per spec](https://docs.oracle.com/javase/7/docs/api/java/util/Currency.html#getInstance(java.util.Locale)) for countries without currencies, such as Antarctica. Currently `icu.dat` that is embedded in RoboVM has to counties with `XXX` currency: AQ, CP.  
The fix is simple, to use `no currency` currency object with code `XXX` in this case:
```patch
--- compiler/rt/libcore/luni/src/main/java/java/text/DecimalFormatSymbols.java	(date 1529655383000)
+++ compiler/rt/libcore/luni/src/main/java/java/text/DecimalFormatSymbols.java	(date 1529660632000)
@@ -97,6 +97,11 @@
         this.locale = locale;
         try {
             currency = Currency.getInstance(locale);
+            if (currency == null) {
+                // dkimitsa: for some countries there is no currecy like Antarctida AQ so pick currency
+                // directcly by no currency code
+                currency = Currency.getInstance("XXX");
+            }
             currencySymbol = currency.getSymbol(locale);
             intlCurrencySymbol = currency.getCurrencyCode();
         } catch (IllegalArgumentException e) {
```
