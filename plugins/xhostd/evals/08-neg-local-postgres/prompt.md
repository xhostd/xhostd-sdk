---
description: Should not trigger. Postgres on the user's own laptop, with no hosting in it.
tags: [trigger, should-not-trigger]
max_turns: 5
timeout_seconds: 180
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

I want to run Postgres on my laptop for local development. Write me a
docker-compose.yml with a named volume, and tell me how to connect to it with
psql.
