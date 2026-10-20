---
description: Should not trigger. An explanation of a Dockerfile, with no deploy asked for.
tags: [trigger, should-not-trigger]
max_turns: 5
timeout_seconds: 180
allowed_tools: [Skill, Read, Glob, Grep]
runs: 3
---

Explain what each line of this Dockerfile does.

```dockerfile
FROM node:22-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "server.js"]
```
