---
layout: post
title: 'Debugger: Local variable resolution rework, first results'
tags: [debugger, kotlin, soot, jvm]
---
Have completed most of code rework responsible for debug information preparation for debugger and frame local variable resolution during debug.   
# Reasons for this:
- there is no direct link between Java variable and local variable begin generated in LLVM IR code. Previous implementation expect one local per variable. Some times it is not true;
- `Soot` performs split of local variable into multiple at each assignment and not always it is not always possible to pack them back, as result same java variable could be represented different memory location (at different part of code);
- previous implementation was resolving variables base on line number (of code and variable visibility). All this cause broken in case of multi statements lines;
- sometimes compilers put broken debug information in class file (e.g. kotlin) as result it is not valid source for decisions.

# What was changed:
Details will be provided in separate post but some facts are bellow:
<!-- more -->

## Soot
[commit](https://github.com/dkimitsa/soot/commit/6f5bbdecc307e0e90108316982201a3e5b245bcb)  
- All previous code for to packing and resolving variables that was based on debug information was reverted to state as it was in 1.8 RoboVM;
- Added additional variable packing step that allows minimize variable sprit as minimal. It also packs stack variables;
- Added Live slot variable resolver that allows to find all initialized locals for any specific step;

## RoboVM
[branch](https://github.com/dkimitsa/robovm/tree/debugger_local_resolution)  
How locals resolution was working before:
- DebugInformationPlugin was preparing information for each variable in way: line number range it is visible, address in memory to read it;
- Debugger: once there was a halt (step or breakpoint) for each frame it has line number. Debugger had just to filter the list of variables visible and left only one visible at this line number;
Quite simple and most time it was working (except cases in `reasons for this` section);

After rework things changes into following:
- DebugInformationPlugin prepares list of variables visible at each instruction (variable slice) and makes execution address to slice map;
- Hooks channel (application part of debugger): provides in each frame number besides `line number` also `PC` this frame was stopped;
- Debugger: based on execution address to slice map it receives list of variables visible at specific instruction.

## Hacks  
I didn't know the way to attach debug information to instruction(at least with LLVM 3.6) and get it back from DWARF part of object file. So have solved hack way by attaching slot information index as part of `location info` by utilizing `column` field. Anyway it has no meaning for Java.

# What's left
- support kotlin specific `SMAP` debug information is missing and will be added next;
- these changes are subject for testing and mostly buggy;
- there is bunch of debugger related bug report to be reviewed [#309](https://github.com/MobiVM/robovm/issues/309) [#312](https://github.com/MobiVM/robovm/issues/312)
