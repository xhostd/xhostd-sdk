# Recipe: Node API on the app template

## What you get

A Node 22 Express service on an HTTPS URL. The platform builds it into its own
image at deploy time, and starts it as a non-root user at boot.

You write two shell scripts. `install.sh` runs once at build time. `launch.sh`
starts your server. The platform supplies the base image, the Dockerfile, the
port and the TLS.

The example is live at
[recipe-node-docs.xhostd.app](https://recipe-node-docs.xhostd.app/). It is the
four files below, and the transcript in this guide deployed them. The API has
two read-only routes, and there is no data store behind them. Do the recipe on
your own account, and you get the same API under your own name.

## The files

Four files at the repo root.

### package.json

```json
{
  "name": "recipe-node-express",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "dependencies": {
    "express": "4.21.2"
  }
}
```

Pin your dependencies to exact versions. A range means two deploys of the same
commit can install different code.

### server.js

```js
import express from "express";

const app = express();

// The health check probes GET / and needs a 2xx. An API whose routes all
// live under /api will fail the deploy even though the process is running,
// unless it creates the file named by $XHOSTD_READY_FILE instead.
app.get("/", (req, res) => {
  res.json({ ok: true, service: "recipe-node-express", uptime: process.uptime() });
});

app.get("/api/echo", (req, res) => {
  res.json({ echo: req.query.q ?? null });
});

const port = Number(process.env.XHOSTD_HTTP_PORT);
app.listen(port, "0.0.0.0", () => {
  console.log(`listening on 0.0.0.0:${port}`);
});
```

Two lines hold the full contract with the platform. `XHOSTD_HTTP_PORT` is the
port for your socket. Read that variable, and never write a number in the code.
`0.0.0.0` is the address for the bind. The health check and the proxy both reach
your container from outside it. **A server on `localhost` is therefore
unreachable, and the port makes no difference.**

### install.sh

```sh
#!/bin/sh
# Runs at BUILD time, as root. Output is baked into the image, so the
# install cost is paid once per deploy and never at boot.
set -eu

npm install --omit=dev --no-audit --no-fund
```

### launch.sh

```sh
#!/bin/sh
# Runs at BOOT, as the non-root 'app' user. Never install anything here.
# exec so the server is PID 1 and receives stop signals directly.
set -eu

exec node server.js
```

`install.sh` is optional, and `launch.sh` is necessary. This division is the
purpose of the `app` template. Each costly step occurs once, at build time, as
root. The boot then only runs `exec`.

## The deploy

Every app owns a git repo, and **`git push` → `deploy` is the standard path**.
A push sends only the diff. Each edit after the first therefore costs a few
lines, and not the full content of every file through a tool call. Use
`commit_files` in one situation only: the machine has no git. **A push stores
your code, but it does not deploy it.** `deploy` is a separate call, so the
commit that goes live is the commit that you name.

Four steps. The
[static site recipe](https://docs.xhostd.com/guides/recipes-static) gives more
detail on the same four steps, for your first app.

**1. Create the app.** This call makes the `prod` channel, its hostname and its
git repo.

```text
create_app(name="recipe-node", template="app")
→ {"id": "3ea579f7-1770-4e5f-84cb-595dad2a4fa6",
   "name": "recipe-node",
   "template": "app",
   "repo_url": "https://git.xhostd.com/docs/recipe-node.git",
   "channels": [{"id": "937e4d14-965c-4959-9adb-effc1555b5b7",
                 "name": "prod",
                 "hostname": "recipe-node-docs.xhostd.app",
                 "current_sha": null}], ...}
```

Keep the app `id` and the `prod` channel `id`, because each later call takes one
or both.

**2. Get a credential.** The token is your git password, and it is valid for
30 days.

```text
get_credentials()
→ {"token": "xh_...", "username": "docs",
   "expires_at": "2026-08-30T17:26:27Z",
   "scopes": ["blob:*", "deploy:*", "repo:*", "db:*", "channel:*", "stats:read"]}
```

**This recipe shows the HTTPS steps.** Where a shell is available, SSH is the
first git transport, because the private half of the key never enters a tool
call. One registration then covers every app on that machine. The
[git guide](https://docs.xhostd.com/guides/git) holds the transport branch and
the SSH commands.

Put it in the **password** field of the remote URL:
`https://<username>:<token>@git.xhostd.com/<username>/<app>.git`.

**3. Clone, commit and push.** A new xhostd repo is empty, and git reports this.
The warning is correct, and it is not a fault:

```bash
$ git clone https://docs:$XHOSTD_TOKEN@git.xhostd.com/docs/recipe-node.git
Cloning into 'recipe-node'...
warning: You appear to have cloned an empty repository.
```

Write the four files into that directory. Then commit them and push them:

```bash
$ cd recipe-node
$ git add -A
$ git commit -m "express service on the app template"
$ git push origin master
```

Do not commit or paste the token. `$XHOSTD_TOKEN` above is the value from
`get_credentials`. A remote URL with a real token in it stays in `.git/config`,
in your shell history, and in the output of every `git remote -v`.

**4. Deploy the branch.**

```text
deploy(app_name="recipe-node",
       channel="prod",
       ref="master")
→ {"deploy_id": "57c7a687-81a7-46bc-a666-7202594a44aa",
   "channel_id": "937e4d14-965c-4959-9adb-effc1555b5b7",
   "status": "queued"}
```

`ref` is a branch name. xhostd resolves it to the current head of that branch,
so after a push you do not need the sha. `sha` is also permitted, when you want
an exact commit, and `sha` has priority if you give both. The deploy then runs
in the background. Read its progress with `get_deploy_log`: the first line of
its reply states the outcome, and you poll while the status is `queued` or
`running`.

## Verify it

```text
get_deploy_log(app_name="recipe-node",
               channel="prod",
               deploy_id="57c7a687-81a7-46bc-a666-7202594a44aa")
```

This is that deploy. The reply starts with a status header, then the log. The
transcript shortens the ids, and it omits the lines that teach nothing here,
such as the buildkit and npm-notice lines:

```text
deploy 57c7a687 — success (sha f6efff40baac)
started: 2026-10-04T06:10:42.032256Z   finished: 2026-10-04T06:10:56.605659Z

[2026-10-04T06:10:42+00:00] deploy begin id=57c7a687-... channel=937e4d14-... sha=f6efff40...
[2026-10-04T06:10:42+00:00] git_sync ok: synced app=3ea579f7-... channel=937e4d14-... sha=f6efff40...
[2026-10-04T06:10:42+00:00] [build] start sha=f6efff40baac92e30d30dd207272130c8e502372
[2026-10-04T06:10:42+00:00] [build] queued 0s, starting
[2026-10-04T06:10:43+00:00] [build] #5 [1/3] FROM ghcr.io/xhostd/xhost-runtime:node22-py313-...@sha256:...
[2026-10-04T06:10:43+00:00] [build] #5 CACHED
[2026-10-04T06:10:43+00:00] [build] #6 [2/3] COPY --chown=app:app . /app
[2026-10-04T06:10:43+00:00] [build] #7 [3/3] RUN if [ -f /app/install.sh ]; then chmod +x /app/install.sh && cd /app && ./install.sh && rm -f /app/install.sh; fi && chown -R app:app /app
[2026-10-04T06:10:44+00:00] [build] #7 1.660 added 69 packages in 2s
[2026-10-04T06:10:44+00:00] [build] #7 DONE 1.8s
[2026-10-04T06:10:47+00:00] [build] finished in 4s
[2026-10-04T06:10:47+00:00] [build] queue wait 0s, build 5s
[2026-10-04T06:10:47+00:00] [build] image 988.84 MB total, 17.48 MB charged — base xhost-runtime:node22-py313 exempt
[2026-10-04T06:10:47+00:00] channel snapshot marker recorded
[2026-10-04T06:10:48+00:00] start_container template=app
[2026-10-04T06:10:49+00:00] health_check container=0da74984eac5... port=3000 timeout=120.0s
[2026-10-04T06:10:49+00:00] [container] [xhost] starting launch.sh (XHOSTD_HTTP_PORT=3000) ...
[2026-10-04T06:10:50+00:00] health_check ok
[2026-10-04T06:10:50+00:00] [container] listening on 0.0.0.0:3000
[2026-10-04T06:10:50+00:00] caddy ensure_route hostname=recipe-node-docs.xhostd.app upstream=10.77.1.7:32100
[2026-10-04T06:10:50+00:00] stop_and_remove old container=6436d158f94c...
[2026-10-04T06:10:56+00:00] stop_and_remove ok
[2026-10-04T06:10:56+00:00] deploy success
```

Read these five lines with attention.

**`FROM ghcr.io/xhostd/xhost-runtime:node22-py313-...`.** The platform builds
on its own base image, `xhost-runtime:node22-py313`. That image has Node 22 and
Python 3.13 together. You do not select the image, and you do not write a
Dockerfile.

**The `RUN` line is the Dockerfile that the platform writes.** The build copies
your repo to `/app`, with the owner `app:app`. Then `install.sh` runs as root,
so `apt-get` and a global `npm` work. The build then **deletes** `install.sh`,
so it can never run again at boot. The line `added 69 packages in 2s` is your
`npm install` in that step. The step runs after the `COPY` of your source, so a
change to any file runs `npm install` again, and that step takes nothing from
a cache. The full build still took five seconds.

**`image 988.84 MB total, 17.48 MB charged`.** Only the data that your build
adds to the platform base counts against your plan's image-size cap. The base
layers are exempt. A redeploy of the `app` template is therefore cheap.

**`channel snapshot marker recorded`.** This is the automatic database snapshot
before the deploy. Every non-`static` deploy records one, also when you do not
use the database. The marker copies no data. It records the moment that a
restore returns the database to, so it costs nothing here.

**`[xhost] starting launch.sh (XHOSTD_HTTP_PORT=3000)`, then `health_check
ok`.** The platform prints that line, not your code. It gives the port that your
process must bind, immediately before `launch.sh` runs. Your own line
`listening on 0.0.0.0:3000` comes after it. Only after the probe passes does
`caddy ensure_route` point the hostname at the new container. This deploy
replaced a running container, so `stop_and_remove old container` comes after
the route, and the old container served every request until then.

To find out if the app is alive, and to read no log, call `get_runtime_log` with
**no** `command`:

```text
get_runtime_log(app_name="recipe-node", channel="prod")
```

A call with no command starts no container on the host, and it returns only the
status header. The header gives the process state, and the exit code if the
process stopped. It also gives an OOM-kill flag, the restart count, and the
container indices that you can read. Those indices are important after a
redeploy. The platform archives the log of the previous container, and does not
delete it. To read the output of the container that failed, give its
`container_index`, and a `command` such as `tail -n 200 app.log`.

Then the proof, in two commands, against the live demo:

```bash
$ curl -sS https://recipe-node-docs.xhostd.app/
{"ok":true,"service":"recipe-node-express","uptime":725.789936934}

$ curl -sS "https://recipe-node-docs.xhostd.app/api/echo?q=hello"
{"echo":"hello"}
```

The two responses have the type `application/json`, and the first is 66 bytes.
The `uptime` value comes from `process.uptime()` in `server.js`. Your own
request to the demo therefore gives a different number from the transcript. The
container is the same one, and the difference is not a fault.

## Compress your responses

The platform does not compress your app's responses for you, so turn
compression on in the app. A compressed page reaches your visitors sooner and
costs you less bandwidth, and doing it in the app compresses the whole path
rather than one hop of it.

Add the dependency to `package.json`, pinned like every other:

```json
"compression": "1.8.2"
```

Then register it before your routes, so it wraps them:

```js
import compression from "compression";

app.use(compression());
```

It compresses JSON and HTML and skips the types that are already compressed,
such as PNG and JPEG. Compressing those a second time costs CPU and saves
nothing.

One page shape to leave out. If a response renders a signed-in visitor's own
details — their email, an API token, a CSRF field — beside a value taken from
the URL, such as a search term you echo back into the form, serve that route
uncompressed. The compressed size of the two together leaks the secret a
character at a time to a site that can make the visitor's browser fetch the
page. Filter it out with `compression`'s own hook:

```js
app.use(compression({
  filter: (req, res) =>
    !req.path.startsWith("/account") && compression.filter(req, res),
}));
```

Call `compression.filter` in your own filter, as the example does. A `filter`
option REPLACES the default content-type check rather than adding to it, so a
filter that only tests the path turns the PNG and JPEG skipping back off.
`compression.filter` is that default, exported so you can keep it.

Pages that render no signed-in identity are unaffected, and so are pages that
reflect nothing from the URL. Both halves have to be present for it to matter.

## When it goes wrong

### It listens on localhost instead of 0.0.0.0

`app.listen(port)` with no host, or with `"localhost"` or `"127.0.0.1"`, binds
only in the container. The health check then gets no answer. The probe waits its
full 120 seconds, and the deploy fails on the timeout. Your log still reports
that the server started. **Bind `"0.0.0.0"`.**

### There is no route at GET /

This is the most common failure on the first deploy of an `app` template. The
health check asks for `/` on the health port, and it needs a 2xx or a 3xx. An
API with all its routes under `/api` answers 404 there. The deploy then fails,
although the process operates correctly. The app can also create the file with
the name in `$XHOSTD_READY_FILE`, which is the other signal that the probe
accepts. If your app does not create that file, add a route at `/`, also a very
simple one.

### launch.sh installs things

`launch.sh` runs at each container start, as the non-root `app` user, on an
image that the platform built before. An `npm install` there is slow at each
boot. It can also fail on the permissions, and the platform discards its result.
Put each install in `install.sh`.

### launch.sh does not exec

Without `exec`, the shell stays PID 1, and your server is its child. The stop
signals then go to the shell, and the shell does not send them to your server.
Each redeploy therefore waits for the full kill timeout, and the server does not
stop correctly. Always use `exec` for the last command.

### The port is hardcoded

`app.listen(3000)` works, but it is still incorrect. The platform gives you the
port in `XHOSTD_HTTP_PORT`, and the platform selects the value. Read the
variable.

### A client-side route 404s on refresh

A single-page app serves one `index.html`, and a client-side router
interprets paths such as `/dashboard`. On the host that such an app migrates
from, a rewrite — nginx `try_files`, or a platform's automatic SPA fallback —
serves `index.html` for every unknown path. xhostd has no such rewrite. The
edge proxies every path straight to your server, so Express must serve
`index.html` for those paths itself. Without the fallback, a refresh or a
deep link on a client-side route answers Express's own 404. The platform
sign-in return — a real HTTP navigation back to the path the visitor was on —
404s for the same reason. Register the API routes first, then end with a
catch-all:

```js
import path from "node:path";

// The API routes are registered before this point, so they win over the
// catch-all.
app.use(express.static("dist"));

app.get(/.*/, (req, res) => {
  res.sendFile(path.resolve("dist/index.html"));
});
```

The section "Single-page apps: the fallback is yours" in
[the Docker recipe](https://docs.xhostd.com/guides/recipes-docker) explains
the failure in full.

The Python equivalent of this recipe is
[Recipe: Python API on the app template](https://docs.xhostd.com/guides/recipes-app-python).
If your app has no server-side code, the
[static site recipe](https://docs.xhostd.com/guides/recipes-static) is simpler.
