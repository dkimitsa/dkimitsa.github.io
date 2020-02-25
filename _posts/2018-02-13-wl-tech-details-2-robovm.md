---
layout: post
title: "W/L TechNotes 2: xib2nib"
tags: ["linux windows", "technote", xib2nib, tutorial, hacking]
---
UIKit xib/storyboard files have be compiled from XML text format to binary NIB. In native Mac environment there is [ibtool](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/ibtool.1.html) as part of XCode toolchain that handle this task. Windows/Linux lack this tool and without it not possible compiling iOS application that contains xib/stroryboard layouts. Lucky for us such tool was found in [Microsoft/WinObjC project](https://github.com/Microsoft/WinObjC) but unfortunately it was not working from scratch. Getting it to so-so working level cost me about 100+ hours to fix showstopper bugs and develop tools that simplifies reverse engineering of nib. Details are bellow:  
<!-- more -->

## Other posts in series
1. [making RoboVM cross-platform]({{ site.baseurl }}{% post_url 2017-12-14-wl-tech-details-1-robovm %})
2. xib2nib (you reading it)
3. Codesign
4. Linker and tools
5. actool and
6. Pain and tears :)

## Why picking up not working tool
Even the tool looked bad and was not able to produce anything useful it was biggest project on subject I saw. There was XML parsing working. These was NIB Writer in place, not working at moment.  There was widest coverage of UI primitives, mostly not working but good to start with. After one day of playing with it I was able to create simple NIB iOS was not crashing at and it was GO sign.

## Other pitfalls
Beside the fact tool is not working out of box it also full of unsecure code, while being cpp project it widely uses stdc libary, half of the code in project is dedicated to parsing of outdated and abandoned xib format from pre Xcode 5 era (check [this link for details](http://blog.ittybittyapps.com/blog/2013/09/19/upgrade-all-the-xibs/)). Also it is part of big `WinObjC` and it is good idea to rip everything else away (kept it for a while to keep original tree just in case PR to upstream). And of course hundreds of compilation warnings goes together.

## The goal was to make this tool able to compile huge project
All my changes are available on github, check this tree [dkimitsa/WinObjC](https://github.com/dkimitsa/WinObjC/tree/xib2nib).
Before exposing [first Linux/Windows proof of concept]({{ site.baseurl }}{% post_url 2017-12-12-robovm-now-linux-windows %}) it was fixed just a little simple to allow primitive xib files to be used(like empty view with label and button).
Later I focused on adding more live to it and tried to use it with pretty big UIKit project - about 150 xib files, all sort of controls. It was a total failure: crashes, collapsed layouts, nothing was working.

To achieve this long ours were spent (100+), lot was fixed and some tools and tutorials were crafted. List of changes ([commits from github](https://github.com/dkimitsa/WinObjC/commits/xib2nib)):
```
33fb90407 UISwitch: added parsing of on/off state
65aa23c91 UIControl fixed to parse enabled/selected/highlighted attributes UISwitch outputs enabled/selected/highlighted attributes
ed0b2a571 Fixed UIView.contentStretch was not parsed from xib
30c64fdeb UISegmentedControl fixed
f0479b964 UIPoint is updated with isValid method, also UIView changed to process only valid rects which removes dummy frame(0,0,0,0) outputs
299a8b812 added UICollectionViewFlowLayout which is required for UICollectionView even basic operations
ddc16b347 UITableView: rowHeight is now being read out
2db8d5dbd optimized structures (UIRect etc), UIEdgeInsets moved to common location
21ebd8774 added support for UITextInputTraits which enables lot of configuration of UITextField (like content type, keyboard type etc) but important one is secure content -- for passwords
f007f4268 UIButton: added support for Tint color
837b76f93 NSLayoutConstraint: fixed multiplier parsing if it present in form of relation e.g. "100:200"
8028e43e7 fixed hugging priority and compression resistance as these are handled different way now
1b94ed585 Changes to skip NSLayoutConstaint marked as placeholder
c3256f7e9 NSLayoutConstraint -- relation was not parsed
031829071 fixed UITableViewCell: it wasn't properly read for storyboard parsing
07be4224f fixed priority read out for NSLayoutConstraint
074ef6f7e fixed bug: multiplier was not initialized in constraint and was corrupting priority
ee63d2bbe UILineBreakMode attribute reworked in UILabel, also it was added to UIButton
84094766a Added UIAdjustsFontSizeToFit=true when there is any adjustment as otherwise apple doesn't read UIMinimumFontSize and UIMinimumScaleFactor
b837b8d31 NSLayoutConstraint fixed to use Double values as apple doesn't accept floats
cbd75ca7d Work on warnings as there was many of these in XCode which was very distractive
5a2083bef Massive changes: 1) added support for UILayoutGuide to support safeArea of iOS11 2) added command line parsing 3) added support for minimum deployment target to be able to produce different output 4) if there is some functionality is not supported due min_dep_version nibs are now exported to folder with two files runtime.nib and unlimited objects-11.0+.nib 5) added support for customModule altogether to customClass. It is required to mange proper swift class name
1c64b56b2 Storyboard segue: added limited support for all types of segue, fixed double entries in nib, multiple time regeneration of on VC and other bugs
e2fed3549 UIColor: added NSColorSpace and friends required by iOS to accept color
00c111d32 added: support for key-value pairs and user defined attributes
50b7e6eaa fixed: UIColor copy creation (instead of copy self was modified)
86839d768 fixed: wrong limit check when writing integers fixed: not saving real values XIBObjectNumber moved to separate location and extended with extra types
b361e8d44 fixed: structs migrated from float to double as in recent nib
259556641 fixed2: connections now handled in one place: ObjectConverter otherwise it resulted in duplicate ones fixed2: UIOutletCollection is now one to many map(as in recent nib) instead of n entries
10d298951 fixed: class entry and class name output to nib
97a7eed0d fixed: written proper NIB header values
b8ccb3ffb fixed: nil object type

```

## is it perfect now ?
It is big chance that it will fail on first Xib of yours as I touched only areas that were in my big project. There was no check sheet of all controls created as tools was too bad and goad was to bring it alive. Now it is ready for per-control check and fix. Personally I don't plan spending extra time on this tool without extra need. There is a tutorial next paragraph that will help newcomers to fix things (it is just a routine work). In any case feel free [opening issue](https://github.com/dkimitsa/WinObjC/issues) on project's Github page.

# Tutorial
Nib file format is not officially documented but it is quite simple one. In general it just keeps key-value data in binary format (very simplified assumption). In any case there are several places in web where it was reverse engineered and and documented. Only few I was used:
- [monoobjc](http://www.monobjc.net/xib-file-format.html)
- [matsmattsson/nibsqueeze](https://github.com/matsmattsson/nibsqueeze/blob/master/NibArchive.md)

## There is no spec what data to be put in NIB file for specific control
This information not is available. All fixes done by me is the result of reverse engineering of nib files produced by native Mac  [ibtool](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man1/ibtool.1.html).
To add new functionality or fix annoying bug following scheme was used:
- create reference .XIB file with UI control under test;
- compile it using `ibtool`;
- compile it using `xib2nib`;
- find the difference and apply fixed

## Tools that helps reverse engineering
There is no official tool from apple. Most useful tool I have found is [davidquesada/ibtool](https://github.com/davidquesada/ibtool). In particular `ibdump.py`. It dumps content of nib file into text output that represents low level format of NIB file:
```
Prefix: NIBArchive
Headers: 1
  0: NSObject
	UINibTopLevelObjectsKey = @1
	UINibObjectsKey = @16
	UINibConnectionsKey = @17
	UINibVisibleWindowsKey = @18
	UINibAccessibilityConfigurationsKey = @19
	UINibKeyValuePairsKey = @20
  1: NSArray
	NSInlinedValue = True
	UINibEncoderEmptyKey = @2
	UINibEncoderEmptyKey = @4
	UINibEncoderEmptyKey = @6
  2: UIProxyObject
	UIProxiedObjectIdentifier = @3
  3: NSString
	NS.bytes = IBFilesOwner
  4: UIProxyObject
	UIProxiedObjectIdentifier = @5
  5: NSString
	NS.bytes = IBFirstResponder
  6: UIView
	UIBounds = (0.0, 0.0, 375.0, 667.0)
	UICenter = (187.5, 333.5)
	UISubviews = @7
	UIAutoresizeSubviews = True
	UIAutoresizingMask = 18
	UIBackgroundColor = @14
	UIOpaque = True
  7: NSArray
	NSInlinedValue = True
	UINibEncoderEmptyKey = @8
  8: UILabel
	UIShadowOffset = (0.0, -1.0)
	UIText = @9
	UITextColor = None
	UIAdjustsFontSizeToFit = False
	UIFont = @10
	UIBounds = (0.0, 0.0, 375.0, 21.0)
	UICenter = (187.5, 10.5)
	UIAutoresizeSubviews = True
	UIAutoresizingMask = 36
	UIContentMode = 7
	UIUserInteractionDisabled = True
	UIViewDoesNotTranslateAutoresizingMaskIntoConstraints = True
	UIViewContentHuggingPriority = @13
  9: NSString
	NS.bytes = HelloRoboVM
 10: UIFont
	UIFontName = @11
	UIFontFamilyName = @12
	UIFontPointSize = 17.0
	UISystemFont = True
 11: NSString
	NS.bytes = Helvetica
 12: NSString
	NS.bytes =
 13: NSString
	NS.bytes = {251, 251}
 14: UIColor
	UIAlpha = 1.0
	UIRed = 1.0
	UIGreen = 1.0
	UIBlue = 1.0
	NSRGB = @15
	NSColorSpace = 2
	UIColorComponentCount = 4
 15: NSString
	NS.bytes = 1.000 1.000 1.000
 16: NSArray
	NSInlinedValue = True
	UINibEncoderEmptyKey = @2
	UINibEncoderEmptyKey = @4
	UINibEncoderEmptyKey = @6
	UINibEncoderEmptyKey = @8
 17: NSArray
	NSInlinedValue = True
 18: NSArray
	NSInlinedValue = True
 19: NSArray
	NSInlinedValue = True
 20: NSArray
	NSInlinedValue = True
```

It is not very useful when layouts and cases becomes complicated. I create three new tools that provide higher level output of NIB file structure. I have forked `davidquesada/ibtool` into [dkimitsa/ibtool](https://github.com/dkimitsa/ibtool/tree/advanced_ibdump) and all changes are in new [ibdump2.py](https://github.com/dkimitsa/ibtool/blob/advanced_ibdump/ibdump2.py). It provides three output modes.

## Tool 1: Viewer of low level data with mapping to HEX dump
Command line for this mode is following `ibdump2.py --hex=dump.html View.nib`. Can be simplified to `ibdump2.py --hex= View.nib` and result will be written to `View.nib.hexdump.html`. Sample output is following screenshot and [sample file]({{ "/assets/2018/02/13/test.nib.hexdump.html"}}):  
![]({{ "/assets/2018/02/13/hexdump.png"}})

This tools dumps same output as `ibdump.py` but allows to see bytes for each object in hex dump which is useful for following:
- allows to evaluate exact bytes behind object;
- allows to check exact type of object (as in dump string "1" could be int, float, double or even string)

## Tool 2: High level text presentation of NIB objects
Command line for this mode is following `ibdump2.py -t View.nib`. It resolves primitive types into simple objects and produce high level tree structure that is nice for evaluation. Such output is good for using with Diff tool. Which allows to highlight the difference between `reference` ibtool and xib2nib. Sample output is bellow:
```
root:
     NSObject1 {
         UINibTopLevelObjectsKey = [
             proxy: IBFilesOwner
             proxy: IBFirstResponder
             @UIView1
         ]
         UINibObjectsKey = [
             proxy: IBFilesOwner
             proxy: IBFirstResponder
             @UIView1
             @UILabel1
         ]
         UINibConnectionsKey = []
         UINibVisibleWindowsKey = []
         UINibAccessibilityConfigurationsKey = []
         UINibTraitStorageListsKey = []
         UINibKeyValuePairsKey = []
     }
UIColor:
     UIColor1 {
         UISystemColorName = blackColor
         UIColorComponentCount = 2
         UIWhite = 0.0
         UIAlpha = 1.0
         NSWhite = 0
         NSColorSpace = 4
     }
     UIColor2 {
         UIColorComponentCount = 4
         UIRed = 1.0
         UIGreen = 1.0
         UIBlue = 1.0
         UIAlpha = 1.0
         NSRGB = 1 1 1
         NSColorSpace = 2
     }
UIFont:
     UIFont1 {
         UIFontName = .SFUIText
         UIFontDescriptor = @UIFontDescriptor1
         UIFontPointSize = 17.0
         UIFontTextStyleForScaling = None
         UIFontPointSizeForScaling = 0.0
         UIFontMaximumPointSizeAfterScaling = 0.0
         UIFontTraits = 0
         UISystemFont = True
         NSName = .SFUIText
         NSSize = 17.0
     }
UIFontDescriptor:
     UIFontDescriptor1 {
         UIFontDescriptorAttributes = {
             NSCTFontUIUsageAttribute = CTFontRegularUsage
             NSFontSizeAttribute = 17
         }
     }
UILabel:
     UILabel1 {
         UIBounds = (0.0, 0.0, 375.0, 21.0)
         UICenter = (187.5, 10.5)
         UIDeepDrawRect = True
         UIUserInteractionDisabled = True
         UIAutoresizeSubviews = True
         UIAutoresizingMask = 36
         UIContentMode = 7
         UIViewContentHuggingPriority = {251, 251}
         UIViewSemanticContentAttribute = 0
         UIText = HelloRoboVM
         UIFont = @UIFont1
         UITextColor = @UIColor1
         UIShadowOffset = (0.0, -1.0)
         UITextAlignment = 4
         UIDisableUpdateTextColorOnTraitCollectionChange = False
     }
UIView:
     UIView1 {
         UIBounds = (0.0, 0.0, 600.0, 600.0)
         UICenter = (300.0, 300.0)
         UISubviews = [
             @UILabel1
         ]
         UIBackgroundColor = @UIColor2
         UIDeepDrawRect = True
         UIOpaque = True
         UIAutoresizeSubviews = True
         UIAutoresizingMask = 18
         UIViewSemanticContentAttribute = 0
     }
```

## Tool 3: High level HTML tree presentation of NIB objects
Command line for this mode is following `ibdump2.py --html=dump.html View.nib`. Can be simplified to `ibdump2.py --html= View.nib` and result will be written to `View.nib.dump.html`. Sample output is following screenshot and [sample file]({{ "/assets/2018/02/13/test.nib.dump.html"}}):  
![]({{ "/assets/2018/02/13/dump.png"}})

This tools is most useful one. Scenario of usage one is quite simple:
* generate html files for both nibs: reference one and one under test;
* open two dumps side by side and find out difference

# Exercise: fixing UIControl flags in UISwitch (enabled, highlighted, selected)
(all changes bellow present in [commit](https://github.com/dkimitsa/WinObjC/commit/65aa23c91ac7b76af9cfd25fd3966bb5d3dc4405))

Lets create a simple [test.xib]({{ "/assets/2018/02/13/test.xib"}}) that includes 4 UISwitch as root object with following settings in each:
* first: default (enabled, not selected, not highlighted);
* second: selected
* third: highlighted
* forth: disabled

The .xib file contains following (only fields of interest are present here so simplified):
```XML
<objects>
    <switch .../>
    <switch selected="YES" .../>
    <switch highlighted="YES" .../>
    <switch enabled="NO" .../>
</objects>
```

## Finding differences
Back to tools, convert xib to reference and test xib and generate dump.html for both of them:
```
ibtool --compile ibtool.nib --minimum-deployment-target 8.0 test.xib
xib2nib --compile xib2nib.nib --minimum-deployment-target 8.0  test.xib
ibdump2.py --html= ibtool.nib
ibdump2.py --html= xib2nib.nib
```
Open both `ibtool.nib.dump.html` and `xib2nib.nib.dump.html` side-by-side and to find out what is missing in xib2nib result:
![]({{ "/assets/2018/02/13/sidebyside.png"}})

## Fixing UIControl
Selected/Highlighed/Enabled are base properties of UIControl so parsing XIB values shall go to single place:
Definition and initialization:
```cpp
// UIControl.h
class UIControl : public UIView {
public:
    ...
    bool _enabled;
    bool _selected;
    bool _highlighted;
    ...
}

// UIControl.cpp
UIControl::UIControl() {
    ...
    _enabled = true;
    _selected = false;
    _highlighted = false;
}
```

Parsing:
```cpp
// UIControl.cpp
void UIControl::InitFromStory(XIBObject* obj) {
    ...
    const char* attr;
    if ((attr = obj->getAttrAndHandle("enabled"))) {
        if (strcmp(attr, "NO") == 0)
            _enabled = false;
    }

    if ((attr = obj->getAttrAndHandle("selected"))) {
        if (strcmp(attr, "YES") == 0)
            _selected = true;
    }

    if ((attr = obj->getAttrAndHandle("highlighted"))) {
        if (strcmp(attr, "YES") == 0)
            _highlighted = true;
    }
    ...
```

There is no universal way to output these values. As different controls do this with different name and logic. So this task up to inheritor classes.

# Fixing UISwitch
Parsing is done in `UIControl` and `UISwitch` has to output these values to nib. To find out names and logic check side-by-side comparison:
```cpp
void UISwitch::ConvertStaticMappings(NIBWriter* writer, XIBObject* obj) {
    if (_enabled)
        AddBool(writer, "UISwitchEnabled", true);
    else
        AddBool(writer, "UIDisabled", true);

    if (_selected)
        AddBool(writer, "UISelected", true);

    if (_highlighted)
        AddBool(writer, "UIHighlighted", true);

    UIControl::ConvertStaticMappings(writer, obj);
}
```

## Bottom line
xib2nib is mostly broken. There is much more to fix. I did as much as time allowed but this tutorial describes simple approach that allows fixing things faster. In anyway feel free [opening issue](https://github.com/dkimitsa/WinObjC/issues) on project's Github page. The project can be moved forward if there is an interest from Community.
