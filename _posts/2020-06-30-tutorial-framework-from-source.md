---
layout: post
title: 'tutorial: building a framework'
tags: [tutorial]
---
Short guide how to get framework binary to use with RoboVM if it is provided by the vendor.
## Building using Carthage 
> Carthage is intended to be the simplest way to add frameworks to your Cocoa application

Most github hosted projects provide a way to build artifacts using [Carthage](https://github.com/Carthage/Carthage#quick-start). Follow [quick start](https://github.com/Carthage/Carthage#quick-start) guide for installation. Process is following:
* create `Cartfile` and set its content up to project info, e.g.: `github "Alamofire/Alamofire" ~> 4.7.2` 
* run `carthage update` to build. 

Result -- dynamic framework.

## Getting from CocoaPods
> CocoaPods is a dependency manager for Swift and Objective-C Cocoa projects

[CocoaPods](https://cocoapods.org) project is highly intended to be used as part of Xcode project.  
Often vendors provide instructions on how to include their artifacts into project dependencies using CocoaPods. Xcode project is required, follow step to create one and build the pod:  
* create new `iOS - Single View App` project in XCode and exit it;
* create `Podfile` in same folder where `${projectname}.xcodeproj` is located;
* fill it with references from vendor (PersonalizedAdConsent here as example):  
```
platform :ios, '9.0'
target 'pods_tutorial' do
use_frameworks!
use_modular_headers!
pod 'PersonalizedAdConsent'
end   
```
* run `pod install --repo-update` to add dependencies to project;
* `cd Pods`
* build for simulator:
```
xcodebuild -configuration Release -sdk iphonesimulator13.5 -scheme PersonalizedAdConsent build \
           CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
           CONFIGURATION_BUILD_DIR=artifacts_sim/
```
* build for devices:
```
xcodebuild -configuration Release -sdk iphoneos13.5 -scheme PersonalizedAdConsent build \
           CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
           CONFIGURATION_BUILD_DIR=artifacts_device/
```

Few moments here:
* `target 'pods_tutorial' do` specifies the target as it is in your Xcode project;
* `-scheme PersonalizedAdConsent` specifies the scheme added by pods, to list all schemes in project use `xcodebuild -list`;
* `-sdk iphoneos13.5` specifies SDK available in Xcode, use `xcodebuild -showsdks` to list all available;
* `CONFIGURATION_BUILD_DIR=artifacts_sim/` specifies the directory, where to put binaries;
* `use_frameworks!` and `use_modular_headers!` command to build a dynamic framework, if removed it will result in static library.

## Building out of sources 
Clone/download [Google Mobile Ads Consent SDK](https://github.com/googleads/googleads-consent-sdk-ios) as an example:  
* Navigate to folder where `PersonalizedAdConsent.xcodeproj` is located;
* use `xcodebuild -list` to get list of targets -- here is `PersonalizedAdConsent`;
* build same way as in case of cocoapods, simulator:
```
xcodebuild -configuration Release -sdk iphonesimulator13.5 -scheme PersonalizedAdConsent build \
           CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
           CONFIGURATION_BUILD_DIR=artifacts_sim/
```
* build for devices:
```
xcodebuild -configuration Release -sdk iphoneos13.5 -scheme PersonalizedAdConsent build \
           CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
           CONFIGURATION_BUILD_DIR=artifacts_device/
```

Note: this way static library/framework will be build.
