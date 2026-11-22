---
description: Should trigger xhostd-environments. In this case, the user wants a second process beside the web service.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Add a background worker to my xhostd project notes-app that runs python worker.py
next to the web process.
