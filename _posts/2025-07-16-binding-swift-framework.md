---
layout: post
title: 'Tutorial: binding pure Swift framework. CocoaPods case'
tags: [binding, swift, tutorial]
---
RoboVM can call native libraries written in ObjectiveC using [ObjectiveC Runtime](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc). Same is possible for Swift libraries if api is exposed as `@objc`. In this case ObjectiveC compatible API are provided that can be used with RoboVM.   
In case of pure Swift code there is no Objc compatible API RoboVM can work with.  
Same thing affects other tools:
## [Kotlin](https://kotlinlang.org/docs/native-objc-interop.html#usage):
> A Swift library can be used in Kotlin code if its API is exported to Objective-C with @objc. Pure Swift modules are not yet supported.

## [Xamarin](https://learn.microsoft.com/en-us/previous-versions/xamarin/ios/platform/binding-swift/walkthrough)
> If the header doesn't exist or has an incomplete public interface (for example, you don't see classes/members) you have two options:
> -Update the Swift source code to generate the header and mark the required members with @objc attribute
> - Build a proxy framework where you control the public interface and proxy all the calls to underlying framework

In every case it requires API to be available for usage in ObjectiveC world. And it is not -- a ObjC compatible wrapper to be created and used with RoboVM/Kotlin/Xamarin. 
<!-- more -->

# The wrapper
The goal of wrapper is to proxy a call to Swift code from ObjectiveC enabled class, that can be used with RoboVM.
For example, following Swift code:
```swift
public class SwiftFramework {
    public func initialize(apiKey: String)
}
```

shall be wrapped as bellow: 
```swift
@objc(SwiftFrameworkRvm)
public class SwiftFrameworkRvm {
    private var raw: AppTransaction!    
    fileprivate init(raw: SwiftFramework!) {
        super.init()
        self.raw = raw
    }
    @objc public func initialize(apiKey: String) { raw.initialize(apiKey: apiKey)
}
```

All required API/classes shall be wrapped this way. Result code to be built into XCFramework and to be used altogether with Swift framework. 

# Simplifying process with CocoaPods
[CocoaPods](https://cocoapods.org)

CocoaPods will allow automatically build wrapper frameworks as well it will generate Xcode project where wrapper can be easily developed. Also, if target library is part of CocoaPods it will be automatically fetched.
Idea is to create local cocoa pod spec file for wrapper.


## Structure of the projec 
For start here is folder structure used for this project:
```
├── cocoapods
│   ├── Podfile
│   ├── src
│   │   └── DonkeyAdsSDKKitRvm
│   │       ├── Classes
│   │       │   └── DonkeyAdsSDKKitRvm.swift
│   │       ├── DonkeyAdsSDKKit.xcframework
│   │       └── DonkeyAdsSDKKitRvm.podspec
└── src/
    └── main
        ├── bro-gen/donkeyads.yaml
        ├── java/org/robovm/pods/donkeyads/** 
        └── robopods/META-INF/robovm/ios/robovm.xml
```

`cocoapods` contains all related to cocoapods activities: 
- `cocoapods/src/DonkeyAdsSDKKitRvm/DonkeyAdsSDKKit.xcframework` -- source pure swift framework that requires the wrapper;
- `cocoapods/src/DonkeyAdsSDKKitRvm/DonkeyAdsSDKKitRvm.podspec` -- podscpec for DonkeyAdsSDKKitRvm, an ObjC wrapper that will be used with RoboVM;
- `cocoapods/src/DonkeyAdsSDKKitRvm/Classes/*` -- source for `DonkeyAdsSDKKitRvm` wrapper 
- `cocoapods/Podfile` -- pod file that will be used to build `DonkeyAdsSDKKitRvm`.
- `src/main/bro-gen/donkeyads.yaml` -- bro-gen spec that will be used to bind `DonkeyAdsSDKKitRvm` into Java;
- `src/java/org/robovm/pods/donkeyads/**` -- java files will go there 
- `src/robopods/META-INF/robovm/ios/robovm.xml` -- `robovm.xml` settings of Java bindings, framework dependency in particular.


## Writing a local cocoa podspec
(`cocoapods/src/DonkeyAdsSDKKitRvm/DonkeyAdsSDKKitRvm.podspec`)
Local podspec will define an artifact that we can use to write a wrapper and that can be used with cocoa-pods build manager to build it.

In case swift framework is available just as local binary `s.vendored_frameworks` key is used to link with it: 
```
Pod::Spec.new do |s|
  s.name             = 'DonkeyAdsSDKKitRvm'
  s.version          = '2.15.0'
  s.summary          = 'DonkeyAdsSDKKitRvm for RoboVM'
  s.description      = 'RoboVM objc wrapper for DonkeyAdsSDKKit swift code'
  s.homepage         = 'https://github.com/dkimitsa/codesnippets/tree/rare.bindings/rare.bindings/donkeyads'
  s.license          = { :type => 'MIT' }
  s.author           = { 'dkimitsa' => 'demyan.kimitsa@gmail.com' }
  s.source           = { :git => 'https://github.com/dkimitsa/codesnippets.git' }
  s.ios.deployment_target = '12.0'
  s.source_files = 'Classes/**/*'
  s.public_header_files = 'Pod/Classes/**/*.h'
  s.vendored_frameworks = 'DonkeyAdsSDKKit.xcframework'
end
```

and if target framework is available through CocoaPods, `s.vendored_frameworks` to be replaced with `s.dependency 'DonkeyAdsSDKKit', s.version.to_s`.
Important moment here is `s.source_files = 'Classes/**/*` that specifies source files to be compiled as part of `DonkeyAdsSDKKitRvm`.
Wrapper source code will be put here later, but at this moment this localpod can be used to build Xcode project. 

## Using local pod to generate Xcode project
To fetch anything from CocoaPods dummy XCode project is required, where CocoaPods dependencies will be injected. Dummy project file from `alt-pods` will be used for this. And `cocoapods/Podfile` to include `DonkeyAdsSDKKitRvm` from local spec looks as bellow:
```
platform :ios, '17.0'
target 'pods' do
use_frameworks!
use_modular_headers!
  pod 'DonkeyAdsSDKKitRvm', :path => 'src/DonkeyAdsSDKKitRvm'
end
```

Note: `:path => 'src/DonkeyAdsSDKKitRvm'` specifies that local spec is used. 

Now it can be built with sequence of commands:
```
# copy dummy project required to fetch lib using cocoapods
cp -R ../../scripts/fetch_pods.xcodeproj ./

PODNAME=DonkeyAdsSDKKitRvm
pod install 
pod update 
pushd Pods

xcodebuild -configuration Release -sdk iphoneos -scheme $PODNAME build \
         CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
         CONFIGURATION_BUILD_DIR=../target-ios
xcodebuild -configuration Release -sdk iphonesimulator -scheme $PODNAME build \
         CODE_SIGNING_REQUIRED=NO CODE_SIGNING_ALLOWED=NO \
         CONFIGURATION_BUILD_DIR=../target-ios-sim
popd

xcodebuild -create-xcframework \
    -framework "target-ios-sim/${PODNAME}.framework" \
    -framework "target-ios/${PODNAME}.framework" \
    -output "${PODNAME}.xcframework"
```

It should go without issues and empty `DonkeyAdsSDKKitRvm.xcframework` is created.   
Also `cocoapods/Pods/Pods.xcodeproj` project files is generated that will be used as project to develop a wrapper.

## Setting up XCode project
Once `cocoapods/Pods/Pods.xcodeproj` is opened, it shows small project for pods, where is `DonkeyAdsSDKKitRvm` is listed under `Developement Pods` group.
![]({{ "/assets/2025/07/16/xcode-local-pod.png"}})

For wrapper code `DonkeyAdsSDKKitRvm.swift` is created and added to `DonkeyAdsSDKKitRvm` group, currently it contains only import of original `DonkeyAdsSDKKit`.
At this moment it should properly see `DonkeyAdsSDKKit` framework but compilation is not enabled as no scheme is selected in `Manage schemes...` dialog,  `DonkeyAdsSDKKitRvm` has to be selected there.

![]({{ "/assets/2025/07/16/xcode-local-pod-scheme.png"}})

After this Xcode should be able to compile project and `DonkeyAdsSDKKitRvm.swift`.

## Writing the wrapper 
First start is to open framework definition (`DonkeyAdsSDKKit`) and copy and paste everything into own source file (`DonkeyAdsSDKKitRvm.swift`).
It will not compile but will be a good draft to show all API that has to be wrapped/processed.

### Wrapping structs
Each structure has to be wrapped into ObjC class and inherit from NSObject, from:
```swift
public struct AdAsset : Codable, Equatable {
}
```
to
```swift
@objc(AdAssetRvm) public class AdAssetRvm : NSObject {
```

Also wrapper shall contain reference to wrapped instance and initializer with it:
```swift
@objc(AdAssetRvm) public class AdAssetRvm : NSObject {
    let raw: AdAsset
    init(raw: AdAsset) { self.raw = raw }
}
```

Each property has to be implemented in the way to proxy call to wrapped swift instance from :
```swift
public let url: URL
public let link: URL?
```
to 
```swift
public var url : URL { raw.url }
public var link : URL? { raw.link }
```

In case property is mutable, both setter/getter to be provided, from:
```swift
public var lastDownloadTimestamp: Date?
```
to
```swift
@objc(AdAssetRvm) public class AdAssetRvm : NSObject {
    var raw: AdAsset
    public var lastDownloadTimestamp: Date? {
        get { raw.lastDownloadTimestamp } set { raw.lastDownloadTimestamp = newValue }
    }
}
```
Note, wrapped instance has to be declared using `var` in this case. 


### Wrapping enums
In simple case, when enum without associated value it can be replaced with numeric one, from:
```swift
public enum AssetType : String, Codable {
    case image
    case video
    ...
}
```
to
```swift
@objc public enum AssetTypeRvm: Int {
    case image
    case video
}
```

Usually it will require marshaller for each direction:
```swift
extension DonkeyAdsSDKKit.AdAsset.AssetType {
    func toRvm() -> AdAssetRvm.AssetTypeRvm {
        switch self {
        case .image: AdAssetRvm.AssetTypeRvm.image
        case .video: AdAssetRvm.AssetTypeRvm.video
        default: AdAssetRvm.AssetTypeRvm.unknown
        }
    }
}
```

and corresponding properties has to be converted, from:
```swift
public let type: DonkeyAdsSDKKit.AdAsset.AssetType
```
to
```swift
@objc public var type : AdAssetRvm.AssetTypeRvm { raw.type.toRvm() }
```

### Wrapping enums (with associated values)
(example from [storekit2](https://github.com/dkimitsa/robovm-cocoatouch-swift/tree/main/storekit))
In this case enum can't be replaced with plain int enum:
```swift
public enum PurchaseResult {
    /// The purchase succeeded with a `Transaction`.
    case success(VerificationResult<Transaction>)

    /// The user cancelled the purchase.
    case userCancelled

    /// The purchase is pending some user action.
    case pending
}
```

Most direct analogy for it is sealed classes in kotlin. As for Java it can be replaced with following:
```swift
@objc(RvmProduct_PurchaseResult)
public class PurchaseResult: NSObject {
    private override init() {}

    /// The purchase succeeded with a `Transaction`.
    @objc(RvmProduct_PurchaseResult_success)
    public class success: PurchaseResult {
        @objc public let transaction: VerificationResultTransaction
        init(transaction: VerificationResultTransaction) {
            self.transaction = transaction
        }
        
        public override func isEqual(_ object: Any?) -> Bool {
            return if let other = object as? success { self.transaction == other.transaction } else { false }
        }

        public override var hash: Int { return transaction.hashValue }
    }
    
    /// The user cancelled the purchase.
    @objc public static let userCancelled = PurchaseResult()

    /// The purchase is pending some user action.
    @objc public static let unknown = PurchaseResult()
}
```

Where `userCancelled`/`unknown` are singleton values and should be just checked by reference while `success` is a data class with payload.  
Code to convert is straight forward:
```swift
extension Product.PurchaseResult {
    func toRvm() -> RvmProduct.PurchaseResult {
        return switch self {
        case .success(let verificationResult): RvmProduct.PurchaseResult.success(transaction: VerificationResultTransaction(raw: verificationResult))
        case .userCancelled: RvmProduct.PurchaseResult.userCancelled
        case .pending: RvmProduct.PurchaseResult.pending
        @unknown default: RvmProduct.PurchaseResult.unknown
        }
    }
}
```

### Wrapping protocols 
Most likely wrapping protocols will consist of:
- creating `obj-c` compatible copy;
- creating a `obj-c` one to original `switch` one.

One tricky point is if protocol instance is to be returned back to `java/objc` side -- its good to return it from proxy.
```swift
public protocol DonkeyAdsDelegate : AnyObject {
    func adsDidLoad()
    func adDidShow(_ assetId: String, impressionHash: String)
    func adDidClick(_ assetId: String, impressionHash: String?)
    func adDidClose(_ assetId: String, impressionHash: String?)
}
```

is replaceable with: 
```swift
class DonkeyAdsDelegateProxy: DonkeyAdsSDKKit.DonkeyAdsDelegate {
    let rvm: DonkeyAdsDelegateRvm
    init(rvm: DonkeyAdsDelegateRvm) { self.rvm = rvm }
    func adsDidLoad() { rvm.adsDidLoad() }
    func adDidShow(_ assetId: String, impressionHash: String) { rvm.adDidShow(assetId, impressionHash: impressionHash) }
    func adDidClick(_ assetId: String, impressionHash: String?) { rvm.adDidClick(assetId, impressionHash: impressionHash) }   
    func adDidClose(_ assetId: String, impressionHash: String?) { rvm.adDidClose(assetId, impressionHash: impressionHash) }
}

extension DonkeyAdsDelegateRvm {
    func toRaw() -> DonkeyAdsSDKKit.DonkeyAdsDelegate { DonkeyAdsDelegateProxy(rvm: self) }
}
```

and is being used as: 
```swift
@objc public class AdSdkRvm: NSObject {
    @objc public static func setDelegate(_ delegate: (any DonkeyAdsDelegateRvm)?) { AdSdk.setDelegate(delegate?.toRaw()) }
}
```

### Tricky case: not exposed obj-c classes
Pure-swift framework might not deliver any `.c`/`.objc` header files while contain proper ObjC subclasses (ex `UIView`/`UIViewController`).
Simplest way to expose it with own wrapper framework is to manually declare them in header file:
```swift
@MainActor @objc @preconcurrency public class BaseQuestionCard : UIView, DonkeyAdsSDKKit.QuestionView {
   @MainActor @preconcurrency weak public var delegate: (any DonkeyAdsSDKKit.QuestionViewDelegate)?
   ...
}
```
stub header file to expose it:
```c
SWIFT_RUNTIME_NAME("_TtC15DonkeyAdsSDKKit16BaseQuestionCard")
@interface BaseQuestionCard : UIView
@end
```

While it is objc class but it will keep swift one runtime, so it should be annotated with mangled name using `SWIFT_RUNTIME_NAME` annotation. 

### Tricky case: extending class while not wrapping it
When it goes for view it might be more comfortable to keep it raw type. In this case `swift` extension / `objc` categories are handy to adding own wrappers to access swift api from objc level:  
```swift
@objc extension DonkeyAdsSDKKit.BaseQuestionCard: QuestionViewRvm {
    public var rvm_delegate: (any QuestionViewDelegateRvm)? {
        get { (delegate as? QuestionViewDelegateProxy)?.rvm }
        set { delegate = newValue?.toRaw() }
    }
    
    public func rvm_configure(question: QuestionRvm, answerState: AnswerStateRvm) {
        configure(question: question.raw, answerState: answerState.raw)
    }
    
    public func rvm_saveAnswer() {
        saveAnswer()
    }
}
```

^^^^ sample shows how to access swift `configure`/`saveAnswer`/`delegate` items by wrapping it with extension. To keep it from name collision items were prefixed with `rvm_`. 


## Building and binding 
Once everything is wrapped framework should be built with script described few pages above. Result can then be bind into java using `bro-gen` (check [tutorial]({{ site.baseurl }}{% post_url 2017-10-19-bro-gen-tutorial %}) ) 

