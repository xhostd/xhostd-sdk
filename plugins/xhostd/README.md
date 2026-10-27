# xhostd plugin

xhostd is hosting for coding agents. With this plugin, your agent creates an
app, pushes its code, and deploys it to an HTTPS URL. It also manages preview
channels, environment variables, secrets, Postgres snapshots, and custom
domains.

## What the plugin contains

- One remote MCP server, `https://mcp.xhostd.com/mcp`.
- The `xhostd` skill and its reference files: worked recipes for each shape
  of app, and the HTTP API reference.
- An icon.

The plugin contains no hooks, scripts, or binaries.

## Install

### Claude Code

Install from the Anthropic directory: run `/plugin`, open **Discover**, and
select **xhostd**.

To install from the marketplace, run these commands:

```
/plugin marketplace add xhostd/xhostd-sdk
/plugin install xhostd@xhostd-sdk
/reload-plugins
```

### Codex

Install from the plugin directory in Codex, or add the marketplace:

```
codex plugin marketplace add xhostd/xhostd-sdk
```

Then run `/plugins`, select **xhostd**, and select **Install plugin**.

### Cursor

Install from the Cursor Marketplace, or from
[cursor.directory](https://cursor.directory): search for **xhostd** and
install it.

To install it as a local plugin, run these commands:

```
git clone https://github.com/xhostd/xhostd-sdk.git
mkdir -p ~/.cursor/plugins/local
cp -r xhostd-sdk/plugins/xhostd ~/.cursor/plugins/local/xhostd
```

Then run **Developer: Reload Window**, open **Customize** in the sidebar, find
**xhostd**, and select **Install**.

### ChatGPT

Find **xhostd** in the ChatGPT apps directory and connect it.

## Start the skill

Your agent loads the skill on its own when you ask it to deploy or host
something.
To start it by hand:

- Claude Code: type `/xhostd:xhostd`.
- Codex: type `$xhostd`.
- Cursor: type `/xhostd`.
- ChatGPT: type `@xhostd`.

## Sign in

The MCP server signs you in with OAuth. The first tool call opens the xhostd
sign-in page in your browser. The page takes a username or email address and
a password, or Google sign-in. Your client holds the OAuth token. The console
[Tokens](https://console.xhostd.com/tokens) page lists every token, and you
can revoke any of them there.

## What the plugin sends, runs, and fetches

- **Tool calls** go to `mcp.xhostd.com`. For each call, xhostd keeps a usage
  count: the day, the account, the tool, and the name of the client software.
  The count holds no arguments, and xhostd deletes it after 90 days.
- **Code** goes to `git.xhostd.com`, the git server of your app's repo.
- **Feedback** goes to the xhostd team only when you ask for it or approve it.
- **A verification email** goes only to an address that you give.
- **Commands** the agent runs on your machine:
  - `git`, to push code.
  - `ssh-keygen`, which reads the fingerprints of the `~/.ssh/xhost_ed25519*`
    keys, and creates `~/.ssh/xhost_ed25519` once, when the file is absent. It creates
    `~/.ssh/xhost_ed25519_<username>` when the first key belongs to another
    xhostd account.
  - `curl`, to `api.xhostd.com`, and to your deployed app's own URL to
    check a deploy.
  - `psql`, to `db.xhostd.com`, only when you ask for it and the app's
    external database access is on.
  - `python3`, to parse JSON.
- **Files** the agent writes, beyond your project: the SSH keys, and, only
  when you ask for object-storage credentials, a file with mode 0600 under
  `~/.xhostd/`.
- **Pages** the agent fetches: `docs.xhostd.com` pages, when it follows a
  link from the skill, and your deployed app's own URL, to check a deploy.

## The XHOSTD_TOKEN variable

Some work happens outside the tools: an HTTPS `git push`, a download link,
and `psql`. That work reads your own xhostd token from the `XHOSTD_TOKEN`
environment variable. You create the token on the console
[Tokens](https://console.xhostd.com/tokens) page and export it yourself.

- The token goes only to `api.xhostd.com`, `git.xhostd.com`, and
  `db.xhostd.com`.
- The skill's own commands read the variable by name, so the token value
  never appears in those commands, and no tool result carries it.
- The git guide also documents a remote with the token in its URL, and
  warns against it: that remote writes the token to `.git/config`.
- A push over SSH needs no token, so on that path you can leave the variable
  unset.

## Support

Read the [support page](https://docs.xhostd.com/support), or email
[support@xhostd.com](mailto:support@xhostd.com).

## Privacy, terms, and license

- [Privacy policy](https://docs.xhostd.com/privacy)
- [Terms of service](https://docs.xhostd.com/terms)
- License: MIT. Read the [LICENSE](LICENSE) file.
