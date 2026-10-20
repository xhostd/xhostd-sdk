---
description: Should trigger. The user names xhostd and asks for a deploy and a public HTTPS URL.
tags: [trigger, should-trigger]
max_turns: 25
timeout_seconds: 600
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Deploy this small Express app with xhostd and give me its public HTTPS URL.

package.json:

```json
{
  "name": "hello-api",
  "version": "1.0.0",
  "main": "server.js",
  "dependencies": { "express": "4.21.2" }
}
```

server.js:

```js
const express = require("express");
const app = express();
app.get("/", (req, res) => res.json({ ok: true }));
app.listen(process.env.PORT || 3000);
```
