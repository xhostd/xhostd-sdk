---
description: Should trigger xhostd-account. In this case, an xhostd tool reports unauthenticated.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Every xhostd tool I call answers that I'm unauthenticated. How do I sign in?
