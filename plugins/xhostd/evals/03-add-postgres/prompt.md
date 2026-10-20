---
description: Should trigger. The user asks for a Postgres database on an existing xhostd app.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

My xhostd app notes-app needs a Postgres database. Add one, and tell me how my
Node code should connect to it.
