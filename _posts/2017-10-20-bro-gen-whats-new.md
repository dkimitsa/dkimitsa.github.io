---
layout: post
title: 'bro-gen: whats new after ios11 binding'
tags: ['bro-gen', whatsnew, binding]
---

bro-gen received several updates and new features during handling iOS11 bindings, mostly fixes but there are new ones. 
<!-- more -->
1. *new* output suggestion yaml after run that allows create bindings [really fast]({{ site.baseurl }}{% post_url 2017-10-20-bro-gen-whats-new %})
2. *new* enum configuration is updated with nserror flag that is used to generate NSError subclass automatically to associate codes with class domain **TODO: add reference to demystified post about NSError subclasses here**
3. *updated* block parameter generation there is no need anymore to typedef blocks parameters
4. *updated* block generation that it can handle argument type of another block inside in block definition
5. *new* parameter to method configuration 'static_constructor_name' that allows to create static creator wrapper when it is not possible to create two constructors due having same arguments configuration. Bellow is a place where this workaround is used:
    ```yaml
    INPriceRange: #since 10.0
        methods:
            '-initWithMaximumPrice:currencyCode:':
                name: initWithMaximumPrice
                static_constructor_name: createWithMaximumPrice
            '-initWithMinimumPrice:currencyCode:':
                name: initWithMinimumPrice
                static_constructor_name: createWithMinimumPrice
    ```
6. *fixed* protocol adapter generation in case it inherits more than one protocol. it was just extending first adapter and other protocol's methods were not implemented
7. *updated* property logic to allow class level properties to be specified (just use + in name, e.g. '+property:'). This is required in some cases when in class there is a class and an instance property with same name. this produces two methods in java intansce one and static with same name. So this options allows to assign different name for either class or instance one

Related: [Tutorial: [ADMOB] using bro-gen to quick generate bindings]({{ site.baseurl }}{% post_url 2017-10-20-bro-gen-whats-new %})
