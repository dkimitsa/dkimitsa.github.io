---
layout: post
title: 'Making BouncyCastleProvider work in IntelliJ Idea Plugin'
tags: ['dirty hack', 'idea-plugin']
---
**UPDATE**: another workarounds explained in [new post]({{ site.baseurl }}{% post_url 2018-07-27-intellij-idea-bouncy-castle-2 %})   
A try to use BC within idea plugging ends with following:
```
Caused by: java.lang.SecurityException: JCE cannot authenticate the provider BC
	at javax.crypto.Cipher.getInstance(Cipher.java:657)
	at javax.crypto.Cipher.getInstance(Cipher.java:596)
	at org.bouncycastle.jcajce.util.NamedJcaJceHelper.createCipher(Unknown Source)
	... 97 more
Caused by: java.util.jar.JarException: Class is on the bootclasspath
	at javax.crypto.JarVerifier.verify(JarVerifier.java:247)
	at javax.crypto.JceSecurity.verifyProviderJar(JceSecurity.java:160)
	at javax.crypto.JceSecurity.getVerificationResult(JceSecurity.java:186)
	at javax.crypto.Cipher.getInstance(Cipher.java:653)
	... 99 more
```
This was happening with RoboVM idea plugin but in general issue is wider. First thing attempt to fix it includes:
<!-- more -->
1. Make sure Idea and Plugin uses compatible version (RoboVM used 1.49), fixed, no luck;
2. BC provider shall not be embedded in any other jar to keep signature. Fixed (was in included in RoboVM Compiler Dist)  

Issue is still there. The only mention was at Idea forum ([How to use BouncyCastle in IntelliJ](https://intellij-support.jetbrains.com/hc/en-us/community/posts/115000451110-How-to-use-BouncyCastle-in-IntelliJ)) without any answer.

So here is an answer:
1. "Class is on the bootclasspath" when there is no `jarURL` provided to `javax.crypto.JarVerifier.verify(JarVerifier.java:247)`
2. `jarURL` is null as `javax.crypto.JceSecurity.getCodeBase` returns null for `BouncyCastleProvider`
3. and this is happening as there is DefaultProtectionDomain attached to BouncyCastleProvider by class loader without any permission list and JAR path

Problem with class loader. Idea loads plugin with own `com.intellij.ide.plugins.cl.com.intellij.ide.plugins.cl.PluginClassLoader` and its superclass `com.intellij.util.lang.UrlClassLoader` which implements ClassLoader and ignores all associated code source and permissions, ones that SecureClassLoader does.

Problem is with Idea itself and it is up to team to fix it  ([IDEA-181010](https://youtrack.jetbrains.com/issue/IDEA-181010))

It is possible to use it anyway with following dirty workaround: loading BCP with class loader.
1. Need a jar to load with `URLClassLoader`, either to use standalone file or get it from class pass as normally it will be included with mvn dependency `URL url =  TestPlugin.class.getClassLoader().getResource("org/bouncycastle/jce/provider/BouncyCastleProvider.class");`
2. Use URLClassLoader to load class, instantiate it

So instead of `Security.addProvider(new BouncyCastleProvider());` use following code:
```java
static {
    Provider bcp = null;
    try {
        ClassLoader cl = TestPlugin.class.getClassLoader();
        URL url =  cl.getResource("org/bouncycastle/jce/provider/BouncyCastleProvider.class");
        if ("jar".equals(url.getProtocol())) {
            url = new URL(url.getPath().substring(0, url.getPath().indexOf('!')));
            cl = new URLClassLoader(new URL[]{url}, null);
            Class cls = cl.loadClass("org.bouncycastle.jce.provider.BouncyCastleProvider");
            bcp = (Provider) cls.newInstance();
        }
    } catch (Exception ignored) {
        // do nothing, bcp will be zero here, probably error message will work
    }
    if (bcp == null) {
        // fallback to normal way, may not work
        bcp = new BouncyCastleProvider();
    }
    Security.addProvider(bcp);
}
```
