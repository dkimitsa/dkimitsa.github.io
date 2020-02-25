---
layout: post
title: "W/L: Check for update enhancement"
tags: ["linux windows", idea]
---
There was basic a check for updates already at compiler level (and it even was [fixed recently](https://github.com/MobiVM/robovm/commit/73012fc5b21c98de898e764dee603249b709aa0f)). But it was providing information only to console. I'm working on automatic toolchain download functionality for Linux/Windows project and it requires version check functionality improvement. Here I describe basic version check that I will propose also for MobiVM current master (it does not contain any Linux/Windows specific parts here).

# Changes visible to user
## Balloon instead of console output
![]({{ "/assets/2018/02/21/update-balloon.png"}})  
<!-- more -->
Code was enhanced to allow high-level code above compiler to display user friendly dialogs instead of console output. In case there is no high level overrides (gradle case) there still will be old style console print.

## Version check in RoboVM menu
![]({{ "/assets/2018/02/21/update-menu.png"}})
Now user can trigger version check at any moment. Also if balloon notification was closed this menu causes same update dialog.

## Update dialog
![]({{ "/assets/2018/02/21/update-popup.png"}})
This dialog is triggered either by "check for update" menu click or by clicking on `updates available` balloon. It contains basic version information and button that will navigate to Download page.

## Other changes
Important one is that plugin can detect updates for Snapshot builds which is often useful.


# Technical details of implementation
All changes are available in separate branch at [dkimitsa/robovm/version_check_enh](https://github.com/dkimitsa/robovm/tree/version_check_enh)

## Support for snapshots
During development snapshots come under same version and to differentiate version `build.timestamp` was added to [META-INF/RoboVM/version.properties](https://github.com/dkimitsa/robovm/blob/84dcee9b248dce1fd5a822df232c0c2df4441d3c/compiler/compiler/src/main/filtered-resources/version.properties#L2):
```
version=${project.version}
build.timestamp=${project.build.timestamp}
```

And timestamp it is just a date in [following format](https://github.com/dkimitsa/robovm/blob/84dcee9b248dce1fd5a822df232c0c2df4441d3c/pom.xml#L36):
```
<properties>
  <maven.build.timestamp.format>yyyyMMdd</maven.build.timestamp.format>
  <project.build.timestamp>${maven.build.timestamp}</project.build.timestamp>
</properties>
```

## Extended information block in `version.json`
To support "what's new" texts and snapshots there were changes to `version.json`. This file is being downloaded (from http://robovm.mobidevelop.com/version) to find out if there is new version is available. Currently is pretty small and contains following:
```json
{"version":"2.3.3"}
```

New one is packed with extra information and bigger. Increased size is main downside of this enhancement:
```json
{
  "release": {
    "version": "2.3.3",
    "description": "2.3.3 release of OOS MobiVM/RoboVM",
    "url": {
      "idea": "http://robovm.mobidevelop.com/downloads/releases/idea/"
    }
  },
  "snapshot": {
    "version": "2.3.4",
    "build.timestamp": 20180228,
    "description": "* Added STDERR flush\n* RoboVM menu moved between [tools] and [vcs]",
    "url": {
      "idea": "http://robovm.mobidevelop.com/downloads/snapshots/idea/"
    }
  }
}
```

Structure of it is following:
* two root keys `release` and `snapshot` which corresponding recent version information;
* both `release` and `snapshot` contain following data:
    - `version` number of today build available;
    - `description` text that will go to "what's new" section;
    - `url` map that contains download URLs for each kind of plugins (e.g. idea/eclipse what ever);
* beside this `snapshot` contains `build.timestamp` to be used to find out if currently used snapshot is outdated.

## Other changes
There is a bunch of changes such as:
- UUID was removed from version check to respect user privacy;
- all version check functionality was moved to [UpdateChecker](https://github.com/dkimitsa/robovm/blob/version_check_enh/compiler/compiler/src/main/java/org/robovm/compiler/util/update/UpdateChecker.java) class;
- introduced [UpdateCheckPlugin](https://github.com/dkimitsa/robovm/blob/version_check_enh/compiler/compiler/src/main/java/org/robovm/compiler/util/update/UpdateCheckPlugin.java) interface and it is being used by `UpdateChecker` as [service](https://github.com/dkimitsa/robovm/blob/version_check_enh/plugins/idea/src/main/resources/META-INF/services/org.robovm.compiler.util.update.UpdateCheckPlugin) to delegate handling of update notification to higher level. Example: version check happens on compiler level but is delegated to Idea plugin where it is displayed as Balloon notification;
- lot of Idea UI changes to display version update dialogs/balloons.
