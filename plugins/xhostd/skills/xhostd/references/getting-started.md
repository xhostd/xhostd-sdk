# Getting Started with xhostd — Worked Example

This walks through what an agent does, end-to-end, when a non-technical user says something like *"build me a little site that lists my favourite coffee shops in Lisbon and put it online"*. Nothing for the user to install, paste or configure — the agent mints the credential and runs the commands.

## Contents

- 0. Prerequisites
- 1. Decide on a name
- 2. Create the app
- 3. Write the site
- 4. Deploy
- 5. Watch the build
- 6. Hand the user the URL
- 7. Iterate
- 8. Optional extras
- Troubleshooting

## 0. Prerequisites

The xhostd connector is connected (claude.ai, the Claude desktop app, or Claude Code through claude.ai), or the xhostd plugin is installed (Claude Code). The xhostd tools are available. This page names each tool by its bare name, such as `deploy`; your client might add a prefix of its own. The user has authenticated once through the browser. If not, in Claude Code, point them to `/mcp` → **claude.ai xhostd** (or **xhostd** when the plugin provides it) → **Authenticate**. In claude.ai or the Claude desktop app, point them to **Settings → Connectors** → **xhostd**, or, if it isn't listed, to the Connectors directory, where they search for xhostd and choose **Connect to Claude**. The sign-in page accepts an xhostd username or email address and password, or Google sign-in. A preprovisioned account signs in directly and requires no signup, username selection, email verification, multifactor authentication (MFA), or SMS. A person who uses Google with an email address that has no xhostd account must choose a **username** on first sign-in (lowercase letters, digits, hyphens; 1–40 chars; cannot start or end with a hyphen). That username becomes part of every public URL. With no person present, the agent registers its own account with an SSH key — [Register as an agent](https://docs.xhostd.com/guides/register-as-agent).

## 1. Decide on a name

Pick a DNS-label app name from the user's description — e.g. `lisbon-coffee`. Confirm the name with them in one line before creating anything. Rules: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`, max 40 chars. Reserved prefixes (rejected by the API): `git`, `api`, `www`, `admin`, `preview`, `staging`.

## 2. Create the app

```
create_app(name="lisbon-coffee", template="static")
```

Use `template="static"` for plain HTML/CSS/JS (served directly). Use `template="app"` if the user wants a dynamic backend — that template runs `install.sh` then `launch.sh` in a runtime with Node 22, Python 3.13, and common build tools. If the project is a single-page app, its server must serve `index.html` for the client-side routes — the platform proxies every path straight through and rewrites nothing.

The response looks like:

```json
{
  "id": "f1e2…",
  "name": "lisbon-coffee",
  "repo_url": "https://git.xhostd.com/alice/lisbon-coffee.git",
  "template": "static",
  "channels": [
    {
      "id": "c0a1…",
      "name": "prod",
      "hostname": "lisbon-coffee-alice.xhostd.app",
      "git_ref_binding": null,
      "current_sha": null,
      "status": "provisioning",
      "pending_deploy": null
    }
  ]
}
```

Every later tool addresses the app by name (`app_name="lisbon-coffee"`) and a channel by name (`channel="prod"`); either also accepts the UUID, on every tool but `add_app_feedback`, which names another owner's app. `git_ref_binding` is `null`: `prod` has no branch until the first deploy with `ref` binds it (or pass `branch` to `create_app`).

## 3. Write the site

Write the files into a working directory, then push them to the app's own repo. **With a shell, `git push`, then `deploy`**, for the first commit and every one after it: a push sends only the diff, so each later edit stays incremental.

The full step list is in **Pushing code with git** in SKILL.md. Pick the path from what the machine can do, before you push — never after a failure. With a shell, **push over SSH**: the private half of the key never enters a tool call, and one registration covers every app on the machine. First compare the fingerprint of every existing `~/.ssh/xhost_ed25519*.pub` with `list_ssh_keys`, and push with the key whose `.pub` file the account lists. If none is listed, make the default keypair in a subprocess if the file is absent, and register the public half once:

```
for f in ~/.ssh/xhost_ed25519*.pub; do ssh-keygen -lf "$f"; done   # compare with list_ssh_keys; if one is listed, push with it
ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519   # only if the file is absent
# then: register_ssh_key(public_key=<content of ~/.ssh/xhost_ed25519.pub>, label="this machine")
git init && git add -A && git commit -m "initial site"
git remote add xhostd-ssh "git@git.xhostd.com:alice/lisbon-coffee.git"
GIT_SSH_COMMAND="ssh -i ~/.ssh/xhost_ed25519 -o IdentitiesOnly=yes" git push xhostd-ssh HEAD:master
```

An SSH push needs no token — the `repo_url` from step 2 gives the username and the app name. `-o IdentitiesOnly=yes` stops ssh from offering another key it finds first.

If `register_ssh_key` answers `409`, the default key belongs to a different xhostd account on this machine. Make `~/.ssh/xhost_ed25519_alice` the same way if it is absent, register it, and push with `GIT_SSH_COMMAND="ssh -i ~/.ssh/xhost_ed25519_alice -o IdentitiesOnly=yes"`. If that registration also answers `409`, this machine holds no usable SSH key for this account: push over HTTPS when `XHOSTD_TOKEN` is set, else use `commit_files`, and never make a key under any other name. A file name proves nothing about the account; only a match in `list_ssh_keys` does.

Use only those two key paths — never the project directory, and never a per-project, per-app or per-tool suffix. They sit in `$HOME`, so every Claude Code session, IDE window and project on the machine reuses one key per account and registers no other.

**HTTPS:** where outbound port 22 is blocked, or the SSH push fails, push over HTTPS with the user's own token from the `XHOSTD_TOKEN` environment variable — one `xh_` secret that is the git password, the Postgres password and the platform API bearer at once. If `XHOSTD_TOKEN` is unset, neither transport can push: use `commit_files`, as the **Without git** paragraph below describes, and never ask the user to paste a token. No tool result carries a token. Send it as a header (`repo_url` came back in step 2), and push:

```
git remote add xhostd "https://git.xhostd.com/alice/lisbon-coffee.git"
git -c http.extraHeader="Authorization: Bearer $XHOSTD_TOKEN" push xhostd HEAD:master
```

You can also mint a scoped, short-lived token with the request that `get_credential_request` describes, and send it the same way from a variable of its own; `references/guide-git.md` shows the commands.

`HEAD:master` on either transport: push under the branch name you then deploy with `ref`, because a fresh `git init` can name the local branch `main`. `deploy(..., ref="master")` in step 4 binds `prod` to `master`. Never write the token value into a command or into a file the user might commit. **Pushing stores the code; it does not deploy** — that is step 4.

**Without git:** in a client with no shell access, or when neither git transport can push, use `commit_files(app_name, message, files)`; with no `ref` it commits to the branch `prod` deploys, or `master` while `prod` has no branch. `files` is a `{path: content-or-null}` map: a string upserts, `null` deletes, and a path you don't name is left alone, so send only what is changing. It returns `{"sha": "abc123…"}`, which is what you then deploy. An app with a connected source repository refuses it, because its code comes from that repository: push to that repository. Worked example: <https://docs.xhostd.com/guides/recipes-commit-files>.

For an `app`-template project the push is the same; the repo needs `install.sh` (optional) and `launch.sh` (required) at its root. The deploy only succeeds if the app signals readiness within 120s, and there are **two ways to do that** — whichever comes first. So the process must:

- **Either serve HTTP:** **bind `0.0.0.0` on `$XHOSTD_HTTP_PORT`** — read `$XHOSTD_HTTP_PORT` from the environment, don't hardcode. (`$PORT` holds the same value for apps that read it; new code reads `$XHOSTD_HTTP_PORT`.) `python app.py` with Flask's default `app.run()` binds `localhost` on a fixed port and will fail the check; pass the host and `$XHOSTD_HTTP_PORT` explicitly — and **answer `/` with a 200**: an API whose routes are all under `/api` fails the check even though it runs; add a minimal `/` handler.
- **Or create `$XHOSTD_READY_FILE`** — for a process with no HTTP surface at all (a queue consumer, a cron-style daemon), create the file at the injected path; the process needs no listener, and the file can be empty. Do it once the work loop is genuinely running, not at the top of `launch.sh`. Such a project keeps its hostname, and that URL returns 502 — expected, not a failure.
- **Boot within 120s and stay within a small memory budget (~128 MB)** at run time — the cap applies to the running server, not to the build.
- **Run as a non-root user.** The container runs as `app`; writable paths are `/app`, `$HOME`, and `/tmp`. All installation must live in `install.sh`, which runs at **build** time as root — installing from `launch.sh` fails with `Permission denied`.

Example minimal pair (`install.sh` bakes deps into the image at build time, then `launch.sh` starts the server at boot):

```sh
# install.sh
#!/bin/sh
set -e
# uv installs the same packages as pip.
uv pip install --system --no-cache flask gunicorn
```
```sh
# launch.sh
#!/bin/sh
set -e
exec gunicorn --bind "0.0.0.0:$XHOSTD_HTTP_PORT" app:app
```

## 4. Deploy

```
deploy(
  app_name="lisbon-coffee",
  channel="prod",
  ref="master",
)
```

Returns `{"deploy_id": "d…", "channel_id": "c0a1…", "status": "queued"}`. Deploys run async. `ref` is a branch name and xhostd resolves it to that branch's current head, so after a push you never need to know the sha. The `ref` also binds `prod` to that branch. To pin an exact commit, pass `sha="abc123…"` — which is what you do with the sha `commit_files` returned; a `sha` deploy binds nothing.

## 5. Watch the build

```
get_logs(app_name="lisbon-coffee", channel="prod", source="deploy", deploy_id="d…")
```

The first line of the reply states the outcome: `deploy <id> — <status> (sha <sha>)`, with status one of `queued`, `running`, `success`, or `failed`. Read the status from that header, not from the log text. Poll while the status is `queued` or `running`; on `failed` the reason is in the log tail. `static` deploys take a few seconds; the first `app`-template deploy takes 30–90 seconds because `install.sh` runs. The channel's `pending_deploy` field in `get_app` also shows the in-flight deploy (`null` once it finishes).

## 6. Hand the user the URL

Read `hostname` from the prod channel (it was in step 2's response, and `get_app` will return it later). Tell the user:

> Live at https://lisbon-coffee-alice.xhostd.app

## 7. Iterate

For each follow-up change — "add a section for tea shops", "fix the broken link" — edit the files in the checkout, `git commit`, `git push xhostd HEAD:master`, then `deploy` with `ref="master"`. The same `prod` channel is targeted, and the remote is already configured, so a follow-up is a couple of shell commands and one tool call. The push carries only the lines that changed. In a client with no shell access, the loop is: `commit_files` — `edits` or `patches` for a file that already exists, `files` for a new one — then `deploy` with the sha it returns.

To preview a change without touching production:

```
create_channel(app_name="lisbon-coffee", name="draft", git_ref_binding="branch:draft")
git push xhostd HEAD:draft
deploy(app_name="lisbon-coffee", channel="draft", ref="draft")
```

The preview is live at `https://draft-lisbon-coffee-alice.xhostd.app`.

## 8. Optional extras

- **Env vars & secrets:** `set_env(app_name, key="LOG_LEVEL", value="debug")` stores a plain value, which passes through the conversation, so never pass it a credential. For a credential, call `set_secret(app_name, key="STRIPE_KEY")`: it takes no value and returns the `PUT` request that uploads it from a file with your own token (`curl -sS --fail-with-body -X PUT -H "Authorization: Bearer $XHOSTD_TOKEN" -H "Content-Type: application/octet-stream" --data-binary @<file> "<url>"`), plus a `console_url` where a user without a shell pastes it. `set_secret(..., generate=True)` generates and stores a random value. A secret is never readable back through MCP; the reveal is a protected action that answers `protected_action` to an agent credential, and it is audit-logged. Add `channel="staging"` to either tool for a per-channel override that wins over the app default at deploy time. `list_env(app_name, channel=…)` shows the resolved view. Every non-static channel automatically has `DATABASE_URL` for its own Postgres database; don't set it. A static channel has no database and no `DATABASE_URL`.
- **Custom domain:** `add_custom_domain(app_name, channel, domain)` returns an `instructions` field with the exact TXT + CNAME/A records the user needs to add at their registrar — relay it verbatim. After they add the records, call `add_custom_domain` again with the same args and `verify=True`. HTTPS works automatically once verified.
- **Google sign-in for parts of the site:** zero-config — no tool to call. `/xhost-auth/*` works on every channel. After sign-in the gateway sets a signed identity cookie `__Host-xhost_id` (an RS256 JWT) on the channel host; the app verifies it against the JWKS at `https://auth.xhostd.com/xhost-auth/jwks` (pin `RS256`, check `iss`/`aud`/`exp`) and gates its own routes. **xhostd does identity only, never edge gatekeeping — nothing is blocked at the edge, so every route stays public (anonymous visitors get `200`) until your app verifies the cookie and enforces access itself in code.** Send signed-out users to `/xhost-auth/login?return_to=<path>`. `__Host-xhost_id` is a reserved cookie name. Per-stack verify snippets: <https://docs.xhostd.com/oauth>.

## Troubleshooting

**Tool reports unauthenticated.** In Claude Code, tell the user to run `/mcp`, select the xhostd entry whose tool reported unauthenticated (**claude.ai xhostd**, **xhostd**, or the plugin's xhostd entry), and choose **Authenticate**. In claude.ai or the Claude desktop app, tell the user to open **Settings → Connectors**, select **xhostd**, and reconnect. If it isn't listed, they search for xhostd in the Connectors directory and choose **Connect to Claude**. On the OAuth path there is no token to set. A registered agent renews its token with its key when the token expired, as [Register as an agent](https://docs.xhostd.com/guides/register-as-agent) describes.

**"app name is already taken"** — they already own one with that name. Pick a different name or delete the old app with `delete_app`.

**"invalid app name"** — name violates the DNS-label rules or starts with a reserved prefix.

**Channel `status: provisioning` after deploy** — deploy is still running. The channel's `pending_deploy` field names it; the first line of `get_logs` with `source="deploy"` gives its status.

**`status: failed`** — read the deploy log; surface the failure to the user in plain language and propose a fix.

**Deploy fails right after start / log says `health check failed for container …`** (app template) — neither readiness signal arrived within 120s: `/` didn't return a 2xx or 3xx on `$XHOSTD_HTTP_PORT` *and* no file was created at `$XHOSTD_READY_FILE`. Usual causes: the server bound `localhost` or a hardcoded port, not `0.0.0.0:$XHOSTD_HTTP_PORT`; there's no `/` route (an API under `/api` only); the boot was too slow; or `launch.sh` hit `Permission denied` — it runs as the non-root `app` user, so installing anything there, or writing outside `/app`/`$HOME`/`/tmp`, crashes it. Fix the bind/`$XHOSTD_HTTP_PORT`, add a `/` handler returning 200, or move the install into `install.sh`. If the project has no HTTP surface at all, create `$XHOSTD_READY_FILE` once its work loop is running; it needs no listener.

**`git push` succeeded but the deploy is empty / nothing changed** — the deploy shipped a different branch from the one you pushed. Deploy with `ref` set to the branch you pushed; that also binds `prod` to it. **`channel_unbound`** — `prod` has no branch yet; deploy with `ref`, or bind it with `update_channel`.
