---
description: Should trigger. The user asks for a preview environment for a branch.
tags: [trigger, should-trigger]
max_turns: 20
timeout_seconds: 450
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Set up a preview channel for the feature/search branch of notes-app, so I can
try the new search before it goes to prod. Give me the preview URL.
