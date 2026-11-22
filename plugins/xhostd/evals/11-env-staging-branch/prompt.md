---
description: Should trigger xhostd-environments. In this case, the user wants an environment bound to a branch.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Create a staging environment on my xhostd project notes-app that deploys the
staging branch.
