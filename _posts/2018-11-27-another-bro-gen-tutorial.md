---
layout: post
title: 'Tutorial: [LGSideMenuController] using bro-gen to quick generate bindings'
tags: ['bro-gen', tutorial, binding]
---
This tutorial mostly copies [old one]({{ site.baseurl }}{% post_url 2017-10-19-bro-gen-tutorial %}), please refer to it for more details.   
It cover quick binding of LGSideMenuController.  
<!-- more -->

[Result yaml file](https://github.com/dkimitsa/codesnippets/blob/master/LGSideMenuController/src/main/bro/LGSideMenuController.yaml)  
[Bindings and sample app](https://github.com/dkimitsa/codesnippets/tree/master/LGSideMenuController)

## Dependencies
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
    src/main/bro-gen  (for yaml file )
    src/main/java     (for generated bindings)
```

## Preparations

Build `LGSideMenuController` using Cartage, here is simple `fetch-binaries.sh` script for this. Run it it root folder of project.:  
```
#/bin/sh

# This script fetches and build the LGSideMenuController framework binary
set -e
mkdir -p libs
rm -f Cartfile
rm -rf Carthage
rm -rf libs/LGSideMenuController.framework
echo 'github "Friend-LGA/LGSideMenuController"' > Cartfile
carthage update
cp -r Carthage/Build/iOS/LGSideMenuController.framework libs/
rm -r Carthage
rm -f Cartfile
rm -f Cartfile.resolved
```

Now time to create template `yaml` file for `bro-gen`. For this it is useful to quick look over hears to get common structure: how function/constant/classes called to find out common prefix that can be used. For `LGSideMenuController` most stuff is started with `LG`. It goes into first run yaml as shown bellow. Put it to `src/main/bro-gen/LGSideMenuController.yaml`:  
```yaml
package: org.robovm.pods.lgp
include: [foundation, uikit, coregraphics]
framework: LGSideMenuController
clang_args: ['-x', 'objective-c']
headers:
    - LGSideMenuControllerFramework.h
typedefs: {}
enums: {}
classes: {}
protocols: {}

functions:
    # Make sure we don't miss any functions if new ones are introduced in a later version
    (LG.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:
    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(LG.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(LG.*):
        class: FIXME
        name: 'Constant__#{g[0]}'
```

NB: `bro-gen` can't handle relative paths and we have to copy `LGSideMenuController.framework` from `libs` to same location as `LGSideMenuController.yaml` (to LGSideMenuController/src/main/bro).

## First run

Run `bro-gen` with command like this (you should be in `LGSideMenuController/src/main/bro` already):  
> ~/projects/git/robovm-bro-gen/bro-gen.rb ../java/ LGSideMenuController.yaml

Where `~/projects/git/robovm-bro-gen/bro-gen.rb` is a path to bro-gen script.

There is expected bunch of output from bro-gen in console and quick bindings relies on `# YAML file pottentialy missing entries suggestions` output, here is an example:
```
console output:
# YAML file pottentialy missing entries suggestions

#enums:
# potentialy missing enums
#enums:
    LGSideMenuAlwaysVisibleOptions: {prefix: LGSideMenuAlwaysVisibleOn}
    LGSideMenuPresentationStyle: {}
    LGSideMenuSwipeGestureArea: {}

# potentialy missing structs
    LGSideMenuSwipeGestureRange: {}

# potentialy missing typedefs
    LGSideMenuSwipeGestureRange: {}

# classes to be updated:
classes:
    LGSideMenuController:
        methods:
            '-initWithRootViewController:':
                name: initWithRootViewController$
... cut
            '+sideMenuControllerWithRootView:leftView:rightView:':
                #trim_after_first_colon: true
                name: sideMenuControllerWithRootView$leftView$rightView$
    LGSideMenuSegue: {}

# protocols to be updated:
protocols:
    LGSideMenuDelegate:
        methods:
            '-willShowLeftView:sideMenuController:':
                #trim_after_first_colon: true
                name: willShowLeftView$sideMenuController$
.. cut                
            '-hideAnimationsBlockForRightView:sideMenuController:duration:':
                #trim_after_first_colon: true
                name: hideAnimationsBlockForRightView$sideMenuController$duration$
```

## Applying suggestion into yaml
Suggestions tell what is missing in yaml file. Also `FIXME` will collect all unhandled global, functions and constant. Lets add all suggestions and missing values. Yaml file looks as bellow now:
```yaml
package: org.robovm.pods.lgp
include: [foundation, uikit, coregraphics]
framework: LGSideMenuController
clang_args: ['-x', 'objective-c']
headers:
    - LGSideMenuControllerFramework.h
typedefs: {}
enums:
    LGSideMenuAlwaysVisibleOptions: {prefix: LGSideMenuAlwaysVisibleOn}
    LGSideMenuPresentationStyle: {}
    LGSideMenuSwipeGestureArea: {}

classes:
    LGSideMenuSwipeGestureRange: {} # struct

    LGSideMenuController:
        methods:
            '-init*:':
                name: init
            '-rootViewWillLayoutSubviewsWithSize:':
                name: rootViewWillLayoutSubviews
            '-leftViewWillLayoutSubviewsWithSize:':
                name: leftViewWillLayoutSubviews
            '-rightViewWillLayoutSubviewsWithSize:':
                name: rightViewWillLayoutSubviews
            '-showLeftViewAnimated:completionHandler:':
                name: showLeftView
            '-hideLeftViewAnimated:completionHandler:':
                name: hideLeftView
            '-toggleLeftViewAnimated:completionHandler:':
                name: toggleLeftView
            '-showLeftViewAnimated:delay:completionHandler:':
                name: showLeftView
            '-hideLeftViewAnimated:delay:completionHandler:':
                name: hideLeftView
            '-toggleLeftViewAnimated:delay:completionHandler:':
                name: toggleLeftView
            '-showRightViewAnimated:completionHandler:':
                name: showRightView
            '-hideRightViewAnimated:completionHandler:':
                name: hideRightView
            '-toggleRightViewAnimated:completionHandler:':
                name: toggleRightView
            '-showRightViewAnimated:delay:completionHandler:':
                name: showRightView
            '-hideRightViewAnimated:delay:completionHandler:':
                name: hideRightView
            '-toggleRightViewAnimated:delay:completionHandler:':
                name: toggleRightView
            '-showHideLeftViewAnimated:completionHandler:':
                name: showHideLeftView
            '-showHideRightViewAnimated:completionHandler:':
                name: showHideRightView
            '-setLeftViewEnabledWithWidth:presentationStyle:alwaysVisibleOptions:':
                name: setLeftViewEnabled
            '-setRightViewEnabledWithWidth:presentationStyle:alwaysVisibleOptions:':
                name: setRightViewEnabled
            '+sideMenuControllerWithRootViewController:':
                exclude: true
            '+sideMenuControllerWithRootViewController:leftViewController:rightViewController:':
                exclude: true
            '+sideMenuControllerWithRootView:':
                exclude: true
            '+sideMenuControllerWithRootView:leftView:rightView:':
                exclude: true
    LGSideMenuSegue: {}

protocols:
    LGSideMenuDelegate:
        methods:
            '-willShowLeftView:sideMenuController:':
                trim_after_first_colon: true
            '-didShowLeftView:sideMenuController:':
                trim_after_first_colon: true
            '-willHideLeftView:sideMenuController:':
                trim_after_first_colon: true
            '-didHideLeftView:sideMenuController:':
                trim_after_first_colon: true
            '-willShowRightView:sideMenuController:':
                trim_after_first_colon: true
            '-didShowRightView:sideMenuController:':
                trim_after_first_colon: true
            '-willHideRightView:sideMenuController:':
                trim_after_first_colon: true
            '-didHideRightView:sideMenuController:':
                trim_after_first_colon: true
            '-showAnimationsForLeftView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-hideAnimationsForLeftView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-showAnimationsForRightView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-hideAnimationsForRightView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-showAnimationsBlockForLeftView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-hideAnimationsBlockForLeftView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-showAnimationsBlockForRightView:sideMenuController:duration:':
                trim_after_first_colon: true
            '-hideAnimationsBlockForRightView:sideMenuController:duration:':
                trim_after_first_colon: true

categories:
    LGSideMenuController@UIViewController: {}

functions:
    LGSideMenuSwipeGestureRangeMake:
        exclude: true # there is a constructor that does exactly same

    # Make sure we don't miss any functions if new ones are introduced in a later version
    (LG.*):
        class: FIXME
        name: 'Function__#{g[0]}'

values:

    # this trap will direct  all notification into LGSideMenuController.Notifications static class
    LGSideMenuController(.*)Notification:
        class: LGSideMenuController
        static_class: Notifications
        name: '#{g[0]}'

    # these are duplicating LGSideMenuController(.*)Notification so just exclude
    kLGSideMenuController(.*)Notification:
        exclude: true

    LGSideMenu(.*)Notification:
        class: LGSideMenuConsts
        static_class: Notifications
        name: '#{g[0]}'

    kLGSideMenu(View|AnimationDuration):
        class: LGSideMenuConsts
        name: 'get#{g[0]}Key'

    LGSideMenuControllerVersionNumber:
        class: LGSideMenuController
        name: getVersionNumber
        readonly: true # it is not marked as const in .h

    LGSideMenuControllerVersionString:
        class: LGSideMenuController
        name: getVersionString
        return_type: '@org.robovm.rt.bro.annotation.Marshaler(StringMarshalers.AsAsciiZMarshaler.class) String'

    # Make sure we don't miss any values if new ones are introduced in a later version
    k?(LG.*):
        class: FIXME
        name: 'Value__#{g[0]}'

constants:
    # Make sure we don't miss any constants if new ones are introduced in a later version
    k?(LG.*):
        class: FIXME
        name: 'Constant__#{g[0]}'
```

## Tricks

-  Classes, protocols: were straight forward. As these sections are copy-paste of suggestions from command line output. Names were corrected to be more java-like. Also there are few interested moments:

- Several class methods were excluded as these duplicates init methods which are mapped to constructors.
```
'+sideMenuControllerWithRootView:leftView:rightView:':
    exclude: true
```

- Notifications values were grouped inside static class `LGSideMenuController.Notifications`, this allows organizing things better:
```
LGSideMenuController(.*)Notification:
    class: LGSideMenuController
    static_class: Notifications
    name: '#{g[0]}'
```

- `UIViewController(LGSideMenuController)` category was mapped into `UIViewControllerExtensions.java` with lines bellow.
```
categories:
    LGSideMenuController@UIViewController: {}
```

- binding of `LGSideMenuSwipeGestureRangeMake` function was excluded as it duplicate constructor of `LGSideMenuSwipeGestureRange`.

- Global value `LGSideMenuControllerVersionNumber` was marked as `readonly` to disable setter generation. it is bug in .h file as it has to be marked as const:
```
LGSideMenuControllerVersionNumber:
       class: LGSideMenuController
       name: getVersionNumber
       readonly: true # it is not marked as const in .h
```

- Special marshaller for char arrays/string values.LGSideMenuControllerVersionString returns array of char. But it is more comfortable to work with it as with `String` than `Pointer`:
```
LGSideMenuControllerVersionString:
    class: LGSideMenuController
    name: getVersionString
    return_type: '@org.robovm.rt.bro.annotation.Marshaler(StringMarshalers.AsAsciiZMarshaler.class) String'
```


## Sample app and bindings
Available in my [github/codesnippet repo](https://github.com/dkimitsa/codesnippets/tree/master/LGSideMenuController)
