---
layout: post
title: 'Making BouncyCastleProvider work in IntelliJ Idea Plugin vol.2'
tags: ['dirty hack', 'idea-plugin']
---
It a follow up of [old story]({{ site.baseurl }}{% post_url 2017-10-22-intellij-idea-bouncy-castle %}). This case hit me back again. This time during a work on keychain utils for linux/windows port. In general this happens in following code:
```java
PKCS12PfxPduBuilder builder = new PKCS12PfxPduBuilder();
...
PKCS12MacCalculatorBuilder macBuilder = new JcePKCS12MacCalculatorBuilder();
PKCS12PfxPdu pfx = builder.build(macBuilder, "".toCharArray());

```
and the reason in following:  
<!-- more -->
```java
// BaseMac.java
protected void engineInit(Key key, AlgorithmParameterSpec  params)
	   throws InvalidKeyException, InvalidAlgorithmParameterException
   {
...
	   if (key instanceof PKCS12Key)
	   {
		...
	   }
```

Here we have variable `key` which contains instance of PKCS12Key but `if (key instanceof PKCS12Key)` doesn't work. This happens as Bouncy Castle code loaded in separate ClassLoader than code that create key variables. And from VMs point of view these are different classes.  

The best case would be to have a fix in Idea [IDEA-181010](https://youtrack.jetbrains.com/issue/IDEA-181010) itself. But as it is not going to happen soon it is time for another workaround:

**It good idea to have `bcprix` library to be loadede with same ClassLoader as `bcprov` library**  
So workaround from [old story]({{ site.baseurl }}{% post_url 2017-10-22-intellij-idea-bouncy-castle %}) is extended to following:
```java
// dkimitsa: this is a dirty workaround to load BC provider under idea,
// details: https://dkimitsa.github.io/2017/10/22/intellij-idea-bouncy-castle/
// bug report: https://youtrack.jetbrains.com/oauth?state=%2Fissue%2FIDEA-181010
Provider bcp = null;
try {
	ClassLoader cl = RoboVmPlugin.class.getClassLoader();
	// dkimitsa: 2018-07-26, modified to include prov and prix jars
	URL urlProv = cl.getResource("org/bouncycastle/jce/provider/BouncyCastleProvider.class");
	URL urlPrix = cl.getResource("org/bouncycastle/pkcs/jcajce/JcePKCS12MacCalculatorBuilder.class");
	if ("jar".equals(urlProv.getProtocol()) && "jar".equals(urlPrix.getProtocol())) {
		urlProv = new URL(urlProv.getPath().substring(0, urlProv.getPath().indexOf('!')));
		urlPrix = new URL(urlPrix.getPath().substring(0, urlPrix.getPath().indexOf('!')));
		cl = new URLClassLoader(new URL[]{urlProv, urlPrix}, null);
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
```


**It is very hard to work with Bouncy Castle from code that is loaded in different ClassLoader than BC**  
It results all the way to "type cast exception" and other side effects (like instanceof testing failure). The solution would be to move user code into BC's class loader contexts. I've just create another class loader that I'm using to load class that works with BC:
```java
ClassLoader bc = Security.getProvider("BC").getClass().getClassLoader();
URL url = HackLoader.class.getClassLoader().getResource(RoboVmPlugin.class.getName().replace('.', '/') + ".class");
if ("file".equals(url.getProtocol())) {
	String s = url.toString();
	url = new URL(s.substring(0, s.indexOf("org/robovm/idea/ikeychain/impl/hack.class")));
}
ClassLoader hackClassLoader = new URLClassLoader(new URL[]{url}, bc);
```

**I moved all BC code to separate class I going to load with `hackClassLoader`**  
like this:
```java
public class Hack implements Consumer<Object[]> {
    @Override
    public void accept(Object[] objects) {
        OutputStream pfxOut = (OutputStream) objects[0];
        X509Certificate cert = (X509Certificate) objects[1];
        PrivateKey key = (PrivateKey) objects[2];
        String keyFriendlyName = (String) objects[3];
        try {
            createPKCS12File(pfxOut, cert, key, keyFriendlyName);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    private void createPKCS12File(OutputStream pfxOut, X509Certificate cert,
	    PrivateKey key, String keyFriendlyName) {
        try {
            PKCS12PfxPduBuilder builder = new PKCS12PfxPduBuilder();
            ...
            pfxOut.write(pfx.getEncoded(ASN1Encoding.DL));
            pfxOut.close();
        } catch (Exception e) {
            throw new RuntimeError(e);
        }
    }
}
```

Few notes here:
*I can't use `Hack` class itself in main code as it will be instantiated from different class loader. So assign I can't assign `Hack h = (Hack)cl.loadClass(Hack.class.getName()).newInstance()` as it will results in type case exception. It needs class/interface that is common for both loaders, like `Consumer` from system class loader;
*Consumer takes only one argument so I pass it by array  

**Usage of this code is trivial**   
Example:
```java
	Consumer<Object[]> ii = (Consumer<Object[]>) hackClassLoader.loadClass(Hack.class.getName()) .newInstance();
	ii.accept(new Object[]{os, cert, matchingCsr.getKeyPair().getPrivate(), matchingCsr.getCommonName()});
```

**Bottom line**   
This ugly workaround is another bad thing we have to do fix [external dependency bug](https://youtrack.jetbrains.com/issue/IDEA-181010). Read [old post]({{ site.baseurl }}{% post_url 2017-10-22-intellij-idea-bouncy-castle %}) to find out reasons.
