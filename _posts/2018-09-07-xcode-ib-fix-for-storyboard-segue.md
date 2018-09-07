---
layout: post
title: 'XCode/IB: fix for invisible segue actions'
tags: [fix, 'interface builder']
---

> Alex @avazquezdev 10:32
I've been following the basic navigation tutorial using storyboard, but I have the problem that it's not possible to adding an Unwind Segue. I select Ctrl-drag from the Cancel button to the exit symbol of the editing scene but the exit symbol is not selectable. Could there be a problem generating the Xcode project? I'd like to make sure it's not a problem with my code. Here you can see the example.title

To have segue action assignments works corresponding objc code shell contains code with action defined as bellow:  
```
/**
 * IBAction unwindToNameList
 */
-(IBAction) unwindToNameList:(UIStoryboardSegue*) sender;
```  

But XCode project generator didn't make difference in parameter types and uses `id` for everything, as result IB doesn't see this method as subject for unwind operations:
```
-(IBAction) unwindToNameList:(id) sender;
```  

The fix is to provide argument type for actions in case of native or custom classes. Changes are available in [PR323](https://github.com/MobiVM/robovm/pull/323)
