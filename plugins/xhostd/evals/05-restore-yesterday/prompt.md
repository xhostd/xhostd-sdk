---
description: Should trigger. The user asks to roll an app's database back to a snapshot.
tags: [trigger, should-trigger]
max_turns: 20
timeout_seconds: 450
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

A bad migration deleted rows in notes-app's prod database this morning. Restore
the prod database to yesterday's nightly snapshot.
