---
description: Should trigger xhostd-git. In this case, the user needs to push to an xhostd repository with port 22 blocked.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Outbound port 22 is blocked on this network. Push my local repository to my xhostd
project notes-app over HTTPS.
