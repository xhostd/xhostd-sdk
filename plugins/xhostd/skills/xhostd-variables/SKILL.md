---
name: xhostd-variables
description: >-
  Stores configuration on an xhostd project. Starts when the user wants a
  variable, a secret, an API key, a password, or the values of a .env file set,
  listed, changed, or deleted on a project, or asks why the app does not see a
  new value. Holds plain variables against secrets, the upload request for a
  secret's value, the notice and pending state after a write, and the reserved
  and injected variable names.
---

# xhostd variables and secrets

A variable is a key and a value that a project's containers read from their environment. A secret is
a kind of variable whose value no tool returns.

## Rules for every xhostd task

1. **Write only on request** (create, delete, push, deploy, restore, expose, change); ask if unsure.
2. **No credential in a plain variable**: store it with `set_secret`, never `set_variable`.
3. **No token value in a command**, URL, file you might commit, or chat: read it from a variable.
4. **No plan advice** or purchase link unasked. On `plan_limit_exceeded`, state the limit; no retry.

## Plain values and secrets

`set_variable` stores a **plain** value, and `delete_variable` deletes a plain variable or a secret.
A plain value passes through the conversation, and `set_variable` warns when the key reads like a
credential. A key that holds a secret refuses a plain value.

For an API key, a token, or a password, call **`set_secret`**. It takes no value: it returns the
`PUT` request that uploads the value, which you run from a shell with the value in a file, as its
description shows, so the value never enters the conversation. With no shell, give the user the
result's `console_url`, which opens the console's add form with the key filled in and "Store as a
secret" checked. `generate=True` stores a random 256-bit value under a new key and returns no
value.

Without `environment` the value is a project-level default; with it, a per-environment override
that wins at deploy time. Every service of an environment receives the same variables.
`list_variables` lists the entries, with each secret's value null.

## A write reaches the app at the next deploy

A write does not restart the app: a running container keeps its variables until the environment's
next deploy. The result of `set_variable`, `delete_variable`, and `set_secret` with `generate=True`
carries xhostd's `notice`, and the upload request answers with one too: relay the `notice` and offer
to deploy the environments it names. `get_project` shows the same state as each environment's
`env_pending`.

## Keys

Keys must match `^[A-Z_][A-Z0-9_]*$`.
Reserved (don't try to set): `XHOSTD_USER`, `XHOSTD_SHA`, `XHOSTD_HTTP_PORT`,
`XHOSTD_FORWARD_PORT`, `XHOSTD_READY_FILE`, `XHOSTD_HOSTNAME`, `XHOSTD_URL`, `XHOSTD_DOMAINS`,
`XHOSTD_PROJECT`, `XHOSTD_ENVIRONMENT`, `XHOSTD_SERVICE`, `XHOST_USER`, `XHOST_SHA`,
`XHOST_HTTP_PORT`, `PORT`, `XHOST_FORWARD_PORT`, `XHOST_READY_FILE`, `DATABASE_URL`,
`DATABASE_URL_READONLY`, `DATABASE_URL_DIRECT`, `DATABASE_HOST`, `DATABASE_PASSWORD`,
`S3_ENDPOINT`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION`. Every other
key that starts with `XHOSTD_` is reserved too.

## Variables the platform injects

The platform sets the reserved keys itself. `DATABASE_URL` and the `S3_*` keys belong to
`xhostd-data`.

The public service of every environment, static included, gets its hostnames: `XHOSTD_HOSTNAME`
(the platform hostname, which the deploy probe sends as `Host`), `XHOSTD_URL` (`https://` plus
that hostname, never a custom domain), and `XHOSTD_DOMAINS` (that hostname, then the full
addresses, then the verified custom domains, comma-separated). Build a host allowlist such as
Django's `ALLOWED_HOSTS` from `XHOSTD_DOMAINS`, never from a hardcoded name. Treat
`XHOSTD_DOMAINS` as a set: don't infer a primary domain from its position, and read the platform
hostname from `XHOSTD_HOSTNAME` or `XHOSTD_URL`. `XHOSTD_PROJECT`, `XHOSTD_ENVIRONMENT`, and
`XHOSTD_SERVICE` carry the project, environment, and service names.
