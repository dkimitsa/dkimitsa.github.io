---
layout: post
title: 'Eclipse plugin: development and updates'
tags: [eclipse, plugin]
date: 2025-10-17 00:00:02
---
Eclipse plugin for RoboVM is being maintained at low priority and there are no plans for major updates in the near future. More over it is not clear if it is being used at all. 
Anyway meanwhile we keep it working and compatible with latest Eclipse versions. Below is the step-by-step instruction on how to set up development environment for it.

<!-- more -->
## Download eclipse 
For plugin development Eclipse IDE for Java Developers is required. It can be downloaded from [Eclipse official website](https://www.eclipse.org/downloads/).

## Install Plug-in Development Environment (PDE)
To install the PDE using the "Install New Software" feature:
1. Open the Install Dialog: menu `Help > Install New Software...`.
2. Select an Update Site: In the "Work with" drop-down list, select the URL for your current Eclipse release (e.g., the "latest releases" site or the specific release name). 
3. Expand the category for General Purpose Tools and check the box for Eclipse Plug-in Development Environment.
4. Complete Installation(review the items, accept the license agreement, and click Finish).
5. Restart Eclipse.

## Import project
1. Open Eclipse and select `File -> Import projects -> Maven -> Existing Maven Projects`
2. navigate to `robovm/plugins/eclipse`
3. make sure `ui`, `feature`, `update-site` are selected

## Hack and test
It should be compilable. To run or debug plugin do the following:
1. Open `Run -> Run Configurations...`
2. Create new configuration of type `Eclipse Application`
3. Set the name, e.g. `RoboVM Plugin Dev`
4. In the `Main` tab, select `Run a product` and choose `org.eclipse.platform.ide`
5. In the `Plug-ins` tab, select `Launch with all workspace and enabled target plug-ins`
6. Click `Apply` and then `Run` or `Debug
