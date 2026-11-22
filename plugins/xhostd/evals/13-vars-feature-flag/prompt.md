---
description: Should trigger xhostd-variables. In this case, the user wants a plain variable set.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Set the variable FEATURE_SEARCH to on for the prod environment of my xhostd
project notes-app.
