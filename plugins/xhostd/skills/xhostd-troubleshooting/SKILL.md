---
name: xhostd-troubleshooting
description: >-
  Diagnoses xhostd deploys and running apps. Starts when a deploy failed and its
  log tail did not explain why, when an app crashes, restarts, or misbehaves
  after a successful deploy, or when an app is slow, answers errors, or an
  environment is unhealthy. Holds the deploy log against the runtime log,
  get_health, the reading order of its findings, and the project's activity feed.
---

# Troubleshoot xhostd deploys and running apps

Diagnosis reads and changes nothing. A fix that deploys, restarts, or rewinds is a write operation:
propose it, and run it when the user asks.

## Rules for every xhostd task

1. **Write only on request** (create, delete, push, deploy, restore, expose, change); ask if unsure.
2. **No credential in a plain variable**: store it with `set_secret`, never `set_variable`.
3. **No token value in a command**, URL, file you might commit, or chat: read it from a variable.
4. **No plan advice** or purchase link unasked. On `plan_limit_exceeded`, state the limit; no retry.

## A failed deploy

When a deploy ends in `status: failed` (the first line of the `get_logs` reply with
`source="deploy"`), the reason is in the log tail:
`health check failed for container … no 2xx/3xx response at GET / … and no readiness file created at $XHOSTD_READY_FILE`
means **neither** signal arrived in time — `/` didn't answer a 2xx or 3xx on `$XHOSTD_HTTP_PORT`
(wrong bind/port, no `/` route, slow boot) and no `$XHOSTD_READY_FILE` was created.
`references/guide-troubleshooting.md` lists the boot-time crashes that stop both signals.

## An app that fails after a successful deploy

When a deploy **succeeds but the app misbehaves later**, the deploy log is the wrong log — it only
covers the build/boot window. Use **`get_logs`** with `project`, `environment`, and
`source="runtime"` for the running app's stdout/stderr; pass `service` to read one service's
container. Start with **no `command`**: the status header alone says whether the container is
running, its exit code, whether it was OOM-killed (your app exceeded the plan's memory limit), and
how many times it restarted. Then pass a `command`, such as `tail -n 200 app.log`, which runs over a
copy of the log. If a redeploy replaced the container, the header lists the readable
`container_index` values, so you can still read why the previous version crashed.

## A slow or unhealthy environment

`get_health` (`project`, `environment`) diagnoses one environment: each service's state in
`services`, and findings that name their service, each with its action. Read `findings` first and
obey each finding's `action`; never infer a retry from the figures.
`references/guide-diagnose-slowness.md` explains the verbs and the blocks.

`list_activity` lists the project's recent events, newest first, so you can see who changed what
before a fault.

## References

- `references/guide-troubleshooting.md` — Troubleshooting: Find build, runtime, traffic, and
  resource evidence.
- `references/guide-diagnose-slowness.md` — Diagnose a slow app: `get_health`, why `findings` comes
  first, the six `action.do` verbs, and what an unavailable block means.
