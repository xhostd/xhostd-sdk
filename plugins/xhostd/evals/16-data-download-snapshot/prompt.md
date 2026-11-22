---
description: Should trigger xhostd-data. In this case, the user wants a database snapshot downloaded.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Download yesterday's snapshot of the prod database of my xhostd project notes-app.
