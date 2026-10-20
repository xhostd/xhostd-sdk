---
description: Should trigger. The user asks to host a static page and does not name xhostd.
tags: [trigger, should-trigger]
max_turns: 25
timeout_seconds: 600
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

I made a one-page landing site for my bakery. Can you host it as a static site
and send me the link?

index.html:

```html
<!doctype html>
<html>
  <head><title>Rosa's Bakery</title></head>
  <body><h1>Rosa's Bakery</h1><p>Fresh bread every morning from 7.</p></body>
</html>
```
