---
layout: post
title: "Alt-pods binding -- useful script"
tags: ['bro-gen', binding, robopods, altpods]
---
Alt-pods binding process includes time-consuming preparation activities like download binaries/copy/update versions etc. An attempt to automate these was made and messy [scripts/dumpdeps.kts](https://github.com/dkimitsa/robovm-robopods/blob/alt/scripts/harvester.kts) was born.  
It automates following flow:
- gives instruction where to download framework to;
- deletes header files in `bro/` folder and copies new;  
- deletes `java/` folder with existing bindings;
- invokes `bro-gen` script to perform binding;
- tries to fetch version information from where possible;   
- updates version information in pom.xml/readme files;
- can process multiple frameworks in multi-thread mode;
  
Also its kotlin-script, and it can be debugged in Idea.

### Usage 
<!-- more -->
Once started without parameters will print help:
```
robovm-robopods % ./scripts/harvester.kts                
Error: not specified framework to process !
usage:
scripts/harvester.kts [--help] [-v] [-p] [framework1 framework2 ...]
   --help, -h : prints this help and exits
           -v : enables verbose output
           -p : enables parallel processing
           -i : interactive mode, will check for existing folder and wait for it
   frameworkX : list of frameworks to process. if not specified all frameworks will be processed
Known frameworks:
    AppCenterAnalytics
    AppCenterCore
    ...
Known gropus:
    AppCenter:
    Facebook:
    ...
```

### Interactive mode 
Useful mode that will check if framework binaries were downloaded and if not will print short instruction:
```
dkimitsa@dkimitsas-Pro robovm-robopods % ./scripts/harvester.kts -i -v Firebase
FirebaseCore.framework:  <<<< starting processing


Missing source header for FirebaseCore.framework at location:
/Users/dkimitsa/Downloads/Firebase/FirebaseAnalytics/FirebaseCore.xcframework/ios-arm64_armv7/FirebaseCore.framework/Headers

Download latest Firebase.zip from https://github.com/firebase/firebase-ios-sdk/releases
Unpack it, expected location /Users/dkimitsa/Downloads/Firebase


Press ENTER key once solved
```

### Binding 
Follow the `bro-gen` output. If there is API changes present corresponding `yaml` file to be processed. 

### Parallel processing 
Frameworks will be processed simultaneously if `-p` argument is specified. This will produce messy output to console and doesn't work if interactive mode `-i` requested.    
Useful if another sanity run required to be processed. 

### Warning -- keep manual code 
Script will remove `java/` folder this means that all data including manually added code will be lost. While `bro-gen` will regenerate most of the code, manually added code need to be merged back. 


