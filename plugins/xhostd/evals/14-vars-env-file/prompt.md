---
description: Should trigger xhostd-variables. In this case, the user wants a .env file's values put on a project.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Put the values from my .env file onto my xhostd project notes-app. The file has
APP_NAME, LOG_LEVEL, and SENDGRID_API_KEY.
