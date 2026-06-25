---
layout: post
title: 'Agent driven alt-pods processing'
tags: [hacking, experiment, binding]
---

While playing with `bulk-updater` agent few frameworks received updates:
* Google Mobile Ads updated from `13.5.0` to `13.6.0`
* Facebook updated from `18.0.3` to `18.1.0`
* IronSource updated from `9.4.1` to `9.4.2`

That was an experiment to implement agent driven alt-pods processing. The goal was to delegate all manual work to sub-agent while keep all deterministic scripts in place.
Following was delegated to agent:
- check upstream for update framework, downloading and unpacking;
- handling new version with existing `harvester` script;
- analyze `bro-gen` suggestions and normalize naming of methods;
- merging of normalized suggestions with existing yaml files;
- restart handling cycle with changes;
- compiling and testing of generated modules;
- allow agent to recovery during normalization and merging.

Somehow it works. Today. But probably will not work tomorrow (check "the bad" section).

### toolset
GitHub Copilot agent was used running as IntelliJ plugin with GPT-5.4-mini model.
All related files located in `.github/` folder of alt-pods repo.

### the good
- automatic fetch of new framework version and storing them at expected location is really working and useful;
- automatic normalization of suggestions and merging with existing yaml files is useful and handles most of the cases;
- doesn't cost too much, around 500 credits for full rebind. Or 200 for update scan with few frameworks updated.

### the bad
- agent is not deterministic, might be changed with any update of the plugin, model;
- most part of agent prompt is to prohibit agent from doing random things;
- lack of proper sandboxing, agent invoke tons of shell commands, generate scripts etc. It is hard to control and verify what it does, as it breaks the idea of automation. So first thing is to do dangerous "allow everything";
- messes with file path. considering "allow everything" it feels bad :)
- slow. rebind cycle takes around 3h, update scan takes around 1h;
- local models like Gemma4 is extremely slow in agent mode (while chat feels okay) and ever more dangerous in messing with file path (like starts looking for files in /root of drive instead of project folder). So it is not usable for this task.


### bottom line
Setup will be used and probably improved in the future, most concern at moment it is slow speed.
