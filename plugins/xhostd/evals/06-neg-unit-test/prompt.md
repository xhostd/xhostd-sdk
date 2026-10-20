---
description: Should not trigger. A local coding task with no hosting in it.
tags: [trigger, should-not-trigger]
max_turns: 5
timeout_seconds: 180
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Write a pytest unit test for this function. Cover an empty string and a string
with punctuation.

```python
import re


def slugify(text: str) -> str:
    return re.sub(r"[^a-z0-9]+", "-", text.lower()).strip("-")
```
