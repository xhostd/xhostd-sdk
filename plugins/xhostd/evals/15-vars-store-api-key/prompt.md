---
description: Should trigger xhostd-variables. In this case, the user wants an API key stored, which must start xhostd-variables and not only the core.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Store my Stripe API key on my xhostd project notes-app so the app can read it.
