---
description: Should trigger xhostd-domains. In this case, the user wants a raw TCP port for a protocol HTTPS cannot carry.
tags: [trigger, should-trigger]
max_turns: 15
timeout_seconds: 300
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

My xhostd project broker runs an MQTT broker. Give it a public TCP port that my
devices can connect to.
