---
name: xhostd
description: >-
  Deploys websites, APIs, workers, and static sites to xhostd and returns a live
  HTTPS URL. Starts when the user wants to deploy or host code, put a site online,
  get a production or preview URL for a project, or check a deploy's status, or
  names xhostd. Holds the deploy flow, the write-safety rules, the runtime
  contract of the static, app, and docker templates, the first step after a
  failed deploy, and the skill for each other xhostd topic.
---

# xhostd hosting

xhostd is hosting designed for agents. You create a project, push its code to the git repo the
project owns, and deploy it. Every project gets a production HTTPS URL; named environments give
preview URLs. A project runs one or more services, such as a web process and a worker. Earlier
versions called a project an app, an environment a channel, and a variable an env var; a document or
skill that uses those names means the new ones. This skill names each tool by its bare name, such
as `deploy`; call the tool whose name ends with it.

The user's instructions take precedence over guidelines provided in a skill.

The model loads this skill on its own for a deploy or hosting request. To start it by hand: in
Claude Code, type `/xhostd:xhostd`; in Codex, `$xhostd`; in Cursor, `/xhostd`; in ChatGPT,
`@xhostd`.

## Rules for every xhostd task

1. **Write only on request.** Creating or deleting projects and services, turning a service off,
   pushing code, deploying, rewinding, restarting, changing variables, restoring databases,
   changing domains, and exposing ports are write operations. Perform them when the user
   explicitly requests the operation and target; when the target or production impact is
   ambiguous, ask before acting. When the user asks to preview or test a change, use a preview
   environment, and never expose a port or restore production data without explicit user
   direction. `delete_project` removes a project and every environment, and cannot be undone.
2. **No credential in a plain variable.** An API key, a token, or a password goes through
   `set_secret`, never `set_variable`.
3. **No token value in a command.** Read a token from `$XHOSTD_TOKEN` or a variable of its own;
   never write its value into a command, a URL, a file you might commit, or the conversation.
4. **No plan advice.** When an operation answers `plan_limit_exceeded`, say the feature or limit
   is unavailable, do not retry, and continue with an available alternative. Do not recommend a
   plan change, show plans, relay a purchase link, or send feedback unless the user asks.

## Other xhostd topics

Start the topic's skill when the task reaches it.

| Topic | Skill |
|---|---|
| The SSH or HTTPS push, keys, git tokens, a connected repository | `xhostd-git` |
| Preview environments, branch bindings, deploy-on-push, services | `xhostd-environments` |
| Variables, secrets, API keys, `.env` values | `xhostd-variables` |
| Postgres, snapshots, restores, file storage, exports | `xhostd-data` |
| Custom domains, raw TCP ports, Google sign-in for visitors | `xhostd-domains` |
| Sign-in, a missing tool, plan limits, usage, feedback to xhostd | `xhostd-account` |
| A failure the deploy log does not explain, crashes, slowness | `xhostd-troubleshooting` |

Other tools: `list_projects` finds a project; `list_source_files` and `read_source_file` read its
repo; `add_note`, `list_notes`, and `vote_note` read and write the project's discussion threads,
which are for the members and never gate a git operation.

## The deploy flow

Create the project, get its code into the repo, deploy.

Read the recipe for your project's shape before step 1: every file, the exact calls, and the
failure modes. Each URL holds the recipe's raw Markdown text; without web access, the runtime
contract below is the minimum.

| Shape | Recipe |
|---|---|
| Static HTML/CSS/JS | <https://docs.xhostd.com/guides/recipes-static.md> |
| Node.js on `app` (Express) | <https://docs.xhostd.com/guides/recipes-app-node.md> |
| Python on `app` (FastAPI) | <https://docs.xhostd.com/guides/recipes-app-python.md> |
| Your own `Dockerfile` | <https://docs.xhostd.com/guides/recipes-docker.md> |
| A background worker with no HTTP | <https://docs.xhostd.com/guides/recipes-worker.md> |
| No shell (`commit_files`) | <https://docs.xhostd.com/guides/recipes-commit-files.md> |

1. **`create_project`** with `name` and `template`: `"static"` for plain HTML/CSS/JS, `"app"` for
   projects with `install.sh`/`launch.sh`, `"docker"` for a `Dockerfile` at the repo root. The
   result holds the project's `repo_url` and its `prod` environment's `hostname`. Later tools take
   the project and environment by name, such as `prod`.
2. **Get the code into the repo.** Pick the path from what the machine can do, before you push:
   - No shell, or no usable SSH key and no `$XHOSTD_TOKEN`: write the commit with `commit_files`.
   - A shell: `git push` to the `repo_url` over SSH, whatever the size of the project.
   - Outbound port 22 blocked, or the SSH push fails: push over HTTPS with `$XHOSTD_TOKEN`.

   `xhostd-git` holds the steps of both transports. **Pushing stores your code; it does not
   deploy.**
3. **`deploy`** with `project`, `environment`, and either `ref` (a branch name; use it after a
   push: it deploys the branch's head and binds the environment to it) or `sha` (an exact commit;
   use it after `commit_files`: it ships once and binds nothing). A `deploy` call that the client
   declines never reached xhostd and queued nothing: tell the user that the deploy did not run,
   and wait for the user to ask for it again.

Then poll **`get_logs`** with `project`, `environment`, `source="deploy"`, and `deploy_id`. The
first line of the reply states the outcome, `deploy <id> — <status> (sha <sha>)`, with status one
of `queued`/`running`/`success`/`failed`/`superseded`/`cancelled`. Poll while the status is
`queued` or `running`. A `static` deploy takes seconds; the first `app` deploy runs `install.sh`
and can take 30–90 s; a `docker` deploy builds the image first.

Give the user the `hostname` that `create_project` or `get_project` returns, and never construct
it. To undo a deploy, call `rewind_deploy`.

**When a deploy ends `failed`**, the reason is in the log tail. A `health check failed` line means
neither readiness signal of the runtime contract arrived in time. When the tail does not explain
the failure, or the app fails after a successful deploy, start `xhostd-troubleshooting`.

## Runtime contract — what makes a deploy succeed

A deploy is only marked **ready** if the app passes a **health check**, and there are **two ways to
pass it** — whichever happens first within the time window:

1. **Answer HTTP.** The platform requests `/` on the app's port with the environment's hostname as
   the `Host` header, and accepts a **2xx or 3xx**. A redirect passes as a 3xx; the platform does
   not follow it. A 4xx or 5xx (a 404 from a missing `/` route, a 500) fails.
2. **Create the readiness file** whose path is in the injected `$XHOSTD_READY_FILE`. It can be
   empty; the platform only checks that it exists.

Use (1) for anything that serves HTTP. Use (2) when the app has no web surface at all, such as a
queue consumer, with no dummy HTTP listener. Signal it once the app is genuinely doing its job,
never as the first line of your start command. Such an environment still gets its URL, which
returns 502 as expected. A service that is not public is ready when it creates the file, or after
10 seconds of running with no restart. Create the file when the service needs an exact signal.

For a public service, if neither signal arrives in time the deploy fails regardless of whether the
app "works." A first deploy often fails for this reason, so design for it up front. An app that
checks the `Host` header builds its allowed hosts from `XHOSTD_DOMAINS`; `xhostd-variables` has the
detail. Every environment of a project that runs code has its own Postgres database: read
`DATABASE_URL` from the environment and never ask the user for a connection string; `xhostd-data`
has the detail.

**`static`** — the committed files are served directly from the **repo root**. Put `index.html` at
the root (it answers `/`). No build runs; commit the final HTML/CSS/JS, not un-built sources. A path
with no committed file behind it answers 404, and nginx does not serve `index.html` for client-side
routes. Health window ~10s.

**`app`** — your process must:
- **Signal readiness one of the two ways.** Serve HTTP on **`0.0.0.0` and the injected
  `$XHOSTD_HTTP_PORT`** (`$PORT` holds the same value) and return a 2xx or 3xx at `GET /`; a pure
  API under `/api` adds a minimal `/` route. Or create `$XHOSTD_READY_FILE` once it is running.
  `references/guide-bkm.md` lists framework defaults that fail the check.
- **Boot within 120s.**
- **Stay within a small memory budget (~128 MB) at run time**, which does not apply to the build.
- **Run as a non-root user.** The container runs as `app`; the writable paths are `/app`, `$HOME`,
  and `/tmp`. Writing anywhere else fails with `Permission denied`.
- **Put ALL installation in `install.sh`, never in `launch.sh`.** `install.sh` runs once at
  **build** time, as root, with a generous memory budget, so a heavy build belongs there.
  `launch.sh` runs at boot as the non-root `app` user, so installing there fails on permissions.

`install.sh` (optional) bakes dependencies into the image at build time; `launch.sh` (required)
execs your long-running server at boot — both from the repo root. Minimal pair (Python):

```sh
# install.sh — runtime deps go here (build time, as root)
#!/bin/sh
set -e
uv pip install --system --no-cache flask gunicorn
```
```sh
# launch.sh — bind 0.0.0.0:$XHOSTD_HTTP_PORT and serve a 200 at "/"
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$XHOSTD_HTTP_PORT" app:app
# node equivalent: exec node server.js  (server.js listens on process.env.XHOSTD_HTTP_PORT, host 0.0.0.0)
```

**`docker`** — the repo has a `Dockerfile` at its **root**; xhostd builds it on every deploy and
runs your image's own `ENTRYPOINT`/`CMD` (no `install.sh`/`launch.sh`). The contract:

- **Signal readiness one of the two ways**, with the same health check as `app`. The file signal
  needs no shell in the image: a distroless worker can `open()` the path from its own code.
- **Env vars are injected at run time only, NEVER as build args.** Never bake a secret into an
  image; read all config from the environment at startup.
- **Run migrations in the image's start command**, not at build time (the database is only reachable
  at run time).
- **Charged image size follows the current account entitlement.** Call `get_stats` with
  `view="overview"` for the current cap. "Charged" = total image size minus warm-base layers; an
  over-cap build fails the deploy with the image size and current cap in the log.
- **Match every `FROM` to a warm base image** — `node:22-slim`, `node:24-slim`, `node:26-slim`,
  `python:3.11-slim`, `python:3.12-slim`, `python:3.13-slim`, `python:3.14-slim`,
  `debian:trixie-slim` — including a build-only stage in a multi-stage build, since every stage that
  names one starts with no pull. The final stage's warm-base layers are also exempt from your
  charged image size.

On both `app` and `docker`, **a single-page app must serve its own `index.html` for every
client-side route**: the platform proxies every path straight to your server and rewrites nothing,
so a refresh, a deep link, or the sign-in return on such a route 404s without it.

## References

- `references/getting-started.md` — An end-to-end worked example.
- `references/api-reference.md` — The underlying HTTP API.
- `references/guide-deployments.md` — Deployments, and results against serving state.
- `references/guide-bkm.md` — Working methods: debugging, stack choices, budgets, the undo path.
- `references/guide-index.md` — The index of the worked recipes.
