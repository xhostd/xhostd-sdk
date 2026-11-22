---
description: Should trigger xhostd-git. In this case, the user's push to an xhostd repository is refused over SSH.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

My git push to my xhostd project notes-app fails with "Permission denied (publickey)".
Get the push working.
