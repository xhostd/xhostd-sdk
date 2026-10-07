# xhostd SDK

Claude Code, Codex, and Cursor plugin for [xhostd](https://xhostd.com) — deploy applications, sites, and services. Push code, get HTTPS URLs.

## Install

```
/plugin marketplace add xhostd/xhostd-sdk
/plugin install xhostd@xhostd-sdk
```

Installing the plugin registers both the xhostd skill and the remote MCP server (`https://mcp.xhostd.com/mcp/`).

## Codex

This repository also includes the Codex plugin manifest at `plugins/xhostd/.codex-plugin/plugin.json`, its OAuth MCP declaration at `plugins/xhostd/.mcp.json`, and a repo-local marketplace at `.agents/plugins/marketplace.json`. The MCP server uses browser-based Google OAuth when a person is present. An agent with no browser registers its own account with an SSH key and adds the server with a bearer header instead; read `plugins/xhostd/skills/xhostd/references/guide-register-as-agent.md`.

After installing, reload plugins in your current session:

```
/reload-plugins
```

## Cursor

This repository also includes the Cursor plugin manifest at `plugins/xhostd/.cursor-plugin/plugin.json` and a Cursor marketplace at `.cursor-plugin/marketplace.json`. The manifest points at the same `plugins/xhostd/.mcp.json` that Codex uses.

Until the plugin is listed in the Cursor Marketplace, install it as a local plugin:

```
git clone https://github.com/xhostd/xhostd-sdk.git
mkdir -p ~/.cursor/plugins/local
cp -r xhostd-sdk/plugins/xhostd ~/.cursor/plugins/local/xhostd
```

Then run **Developer: Reload Window**, open **Customize** in the sidebar, find **xhostd**, and select **Install**. To sign in, turn on the **xhostd** MCP server in Customize; a browser opens for sign-in. A team admin can instead import this repository as a team marketplace, and the plugin then appears in Customize for the team.

## Connect

Run `/mcp`, select **xhostd**, and choose **Authenticate**. Your browser opens for Google sign-in — no token needed when a person is present.

## Usage

Just use `/xhostd` — it handles everything:

```
"deploy my website"          → signs up, creates app, pushes, deploys
"check my app status"        → shows apps, channels, URLs, deploy state
"create a preview for this branch" → pushes branch, creates preview URL
```

Or invoke it explicitly:

```
/xhostd
```

The single `/xhostd` skill handles account setup, app creation, deploys, previews, and status checks. Claude figures out what you need from context.

In Codex and Cursor, describe the task normally or mention the xhostd skill; slash-command syntax is client-specific.

## Example use cases

- Deploy the current website or API to xhostd and return its live HTTPS URL.
- Create a preview channel for the current branch and report its URL and deploy status.
- Inspect apps, channels, runtime logs, environment metadata, domains, or deployment history.

## What xhostd supports

- **Static sites** — nginx serves your HTML/CSS/JS
- **Node.js apps** — Express, Next.js, Fastify, Vite (give `install.sh` + `launch.sh`)
- **Python apps** — FastAPI, Flask, Django (give `install.sh` + `launch.sh`)
- **Docker apps** — any runtime, from a `Dockerfile` that you write
- **Background workers** — processes that run without end, not web servers

## Recipes

<https://docs.xhostd.com/guides> holds one complete recipe for each shape of app: every file, the exact calls, and the failure modes of that shape. Read the recipe for your shape before you write the code. The plugin carries the same recipes offline, under `plugins/xhostd/skills/xhostd/references/`.

## How it works

1. You push code to xhostd's git server
2. You trigger a deploy (explicitly, via `/xhostd` or the API)
3. xhostd runs your `install.sh` (install the dependencies) then `launch.sh` (start the app on `$XHOSTD_HTTP_PORT`)
4. Your app is live over HTTPS with a wildcard cert

A Docker app runs the `CMD` in your `Dockerfile` in place of the two scripts. A static site needs no script, because nginx serves the committed files.

## Requirements

- Git installed locally
- An API token (from [console.xhostd.com/tokens](https://console.xhostd.com/tokens)) only if you push over git or call the API with raw curl — the MCP connection itself uses OAuth, no token; an agent that registered with its SSH key holds the token `POST /registrations` answered instead

## License

MIT
