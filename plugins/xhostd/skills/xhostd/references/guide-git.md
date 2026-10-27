# Push code with git

Every app on xhostd owns a git repo at
`https://git.xhostd.com/<username>/<app>.git`. A push to that repo puts
code onto an app. This applies to the first commit, and to every commit
after it. A push sends only the diff, so each edit stays small. A push stores
your code; it does not deploy the code. Start a build separately with
`deploy`, or with the console.

## Contents

- Pick the path before you push
- Mint credentials
- Push over SSH
- The clone / push URL
- Renew an expired token
- A push fails again and again: clear a stale cached credential
- Connect a GitHub repository with the xhostd GitHub app
- Deploy on each GitHub push
- Notes

## Pick the path before you push

The repo answers two transports, and one MCP tool writes to it directly.
Test the machine first, and pick one of the three paths below. Do not
wait for a failure.

1. **The machine runs no command.** A client with no shell access cannot
   run git. Use the `commit_files` MCP tool.
2. **The machine runs commands.** Push over SSH with a key of the
   account you are connected as. Read **Push over SSH** below. Check
   every `~/.ssh/xhost_ed25519*` key against `list_ssh_keys` first, and
   push with the key that the account lists.
3. **The network blocks outbound port 22, or an SSH push fails.** Push
   over HTTPS with a token: your own token from the `XHOSTD_TOKEN`
   environment variable, or a scoped, short-lived token that you mint
   from it. Read **The clone / push URL** below.
4. **Neither transport can push.** There is no usable SSH key, and
   `XHOSTD_TOKEN` is not set. Use the `commit_files` MCP tool, as on a
   machine with no shell. Never ask the user to paste a token.

With SSH, a private key never enters a tool call, and the push needs no
token. HTTPS needs a token in the environment, and that token expires.

The SSH path costs no extra step in the steady state. Your tool makes a
keypair only when the machine holds no key for the account. A key
belongs to the account and not to one app, so one `register_ssh_key`
call serves every app on that machine.

## Mint credentials

You use one **unified credential**. A single `xh_` token is your git
password over HTTPS, your Postgres password, and your platform API
bearer. It carries the full default scopes: repo, deploy, channel, db,
blob, and stats read. Thus the same token can push code, deploy code,
and manage your apps. An SSH push needs no token; read **Push over SSH**
below.

Keep the token in the `XHOSTD_TOKEN` environment variable, and never
write its value into a command. It comes from one of two places:

- **From the console:** open
  [console.xhostd.com/tokens](https://console.xhostd.com/tokens),
  create a token, and export it as `XHOSTD_TOKEN`.
- **From an account you registered with your key:** the token
  `POST /registrations` answered. Export it as `XHOSTD_TOKEN`. Renew it
  with `POST /auth/ssh-key`; read
  [Register as an agent](https://docs.xhostd.com/guides/register-as-agent).

You can use that token directly, or mint a scoped, short-lived token
from it, and both work the same way. A scoped token suits an unattended
job, or a token that you hand to another process. No MCP tool result
carries a token: the `get_credential_request` MCP tool describes the
`POST /credentials` request, and you run that request with your own
token. Keep the minted token in a variable of its own, so its value
stays out of every command and out of the output:

```sh
XHOSTD_GIT_TOKEN="$(curl -sS --fail-with-body -X POST \
  -H "Authorization: Bearer $XHOSTD_TOKEN" \
  --json '{"scopes":["repo:*"],"expires_in":3600}' \
  https://api.xhostd.com/credentials \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); sys.exit(json.dumps(d)) if "token" not in d else print(d["token"])')" \
  && git -c http.extraHeader="Authorization: Bearer $XHOSTD_GIT_TOKEN" push https://git.xhostd.com/<username>/<app>.git HEAD:master
```

Run the mint and the command that uses the token as one shell command. A
shell variable does not survive into a new shell, and an agent's tool
often starts a new shell for each command. If the mint fails, `python3`
prints the error envelope to stderr and exits, so `&&` stops before the
push.

`"scopes":["repo:*"]` holds the token to git access, and
`"expires_in":3600` ends it after one hour. For Postgres, mint with
`"scopes":["db:*"]` into a variable such as `XHOSTD_DB_TOKEN`, and chain
`&& PGPASSWORD="$XHOSTD_DB_TOKEN" psql ...` the same way.

The token starts with `xh_` and is valid for up to **30 days**. A mint
never outlives its caller, so a token you mint from another token expires
with it: read `expires_at` in the response for the exact time. After it
expires, get a new token.

## Push over SSH

The git host answers SSH, and both transports reach the same repo. This
is the first path on a machine that runs commands. A private SSH key
never enters a tool call, so a content filter has nothing to refuse.

**An SSH push needs no token.** It needs your username and the app name
only. `get_app` and `GET /apps/{app_id}` return the `repo_url` of each
app, and that URL holds both values.

A machine can hold keys for more than one xhostd account, so first
check that the key belongs to the account you are connected as:

1. Get the `repo_url` and `<username>` of the app from `get_app`.
2. Read the fingerprint of every existing `~/.ssh/xhost_ed25519*.pub`
   with `ssh-keygen -lf <file>`: the default `~/.ssh/xhost_ed25519.pub`
   and every `~/.ssh/xhost_ed25519_*.pub`. Compare each with the
   `fingerprint` values that `list_ssh_keys` returns. That tool lists
   the keys of the connected account, so a match proves that the key
   pushes for this account. Push with the key whose `.pub` file
   `list_ssh_keys` lists.
3. If none is listed, register the default key. Make
   `~/.ssh/xhost_ed25519` if it is absent, then call `register_ssh_key`
   with it, and push with it.
4. A `409` means that the default key belongs to a different xhostd
   account on this machine. Make `~/.ssh/xhost_ed25519_<username>` if it
   is absent, register it, and push with it.
5. If that registration also answers `409`, this machine holds no
   usable SSH key for this account. Push over HTTPS when `XHOSTD_TOKEN`
   is set, or else use the `commit_files` tool. Never make a key under
   any other name.

A file name proves nothing about the account. A match in
`list_ssh_keys` is what proves which account a key belongs to.

To make a key, run `ssh-keygen` on the machine that pushes:

```sh
ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519
```

For an account-specific key, replace the path with
`~/.ssh/xhost_ed25519_<username>`. That command writes the private half
straight to the disk. Send the **public** half only — the content of the
`.pub` file. Call the `register_ssh_key` MCP tool with that content. Add
a `label` that identifies the machine. You can also paste that content
on [console.xhostd.com/ssh-keys](https://console.xhostd.com/ssh-keys),
which lists the keys of your account and deletes one.

**Keep every key at `~/.ssh/xhost_ed25519` or
`~/.ssh/xhost_ed25519_<username>`.** Do not put a key in the project
directory. Do not add a suffix for a project, an app, or a tool. Those
paths are in your home directory, so every tool and every project on the
machine reads one key per account.

Never make a second key for the same account. A different path makes a
second keypair. Each new keypair adds a key to your account, and the
platform sends you a notification. Your account then collects keys that
no tool uses.

**A key belongs to the account, not to one app.** Thus one
`register_ssh_key` call serves every app that you push to from that
machine. Repeat this step on a new machine, or for a new account, only.

Then set the SSH remote and push. Replace `<key>` with the private key
that the steps chose, `~/.ssh/xhost_ed25519` or a
`~/.ssh/xhost_ed25519_<username>` key:

```sh
git remote add xhostd-ssh "git@git.xhostd.com:<username>/<app>.git"
GIT_SSH_COMMAND="ssh -i <key> -o IdentitiesOnly=yes" git push xhostd-ssh HEAD:master
```

The `HEAD:master` refspec is deliberate. Push under the name you will
deploy: `deploy(..., ref="master")` deploys that branch and binds the
channel to it. A new local repo often uses `main` as its default branch.
`HEAD:master` pushes the current branch to `master`, whatever its local
name is. `GIT_SSH_COMMAND` names the private half. The key
sits outside a default path, so every SSH push needs that variable.
`-o IdentitiesOnly=yes` stops ssh from offering another key it finds
first.

`list_ssh_keys` returns the keys of your account, newest first. It
returns metadata only: `id`, `label`, `algo`, `fingerprint`,
`created_at` and `last_used_at`. It never returns a key.
`delete_ssh_key` removes one key by `key_id`, and a push with that key
then fails at once.

A fingerprint is unique over the whole platform. Thus a key that the
platform holds already answers `409`. You register only a key that
`list_ssh_keys` does not list, so a `409` means that another xhostd
account holds the key. Never make a key to clear a `409` for a key that
`list_ssh_keys` lists for this account.

An SSH push needs outbound port 22. Some company networks and CI runners
block that port. Two conditions send you to a different path: the
network blocks port 22, or an SSH push fails. Push over HTTPS then, as
the next section describes. If `XHOSTD_TOKEN` is not set, use the
`commit_files` tool.

## The clone / push URL

This is the HTTPS transport. Push over HTTPS where the network blocks
outbound port 22, or where an SSH push fails. It needs a token in
`XHOSTD_TOKEN`. Without one, use the `commit_files` tool, and never ask
the user to paste a token.

An agent sends its own token from `XHOSTD_TOKEN` as a header, so the
token value never appears in a command or in `.git/config`:

```sh
git -c http.extraHeader="Authorization: Bearer $XHOSTD_TOKEN" push https://git.xhostd.com/<username>/<app>.git HEAD:master
```

A scoped token that you mint works the same way from its own variable,
as **Mint credentials** shows.

A person at a terminal can also put the token in the **password** field
of the URL:

```
https://<username>:<token>@git.xhostd.com/<username>/<app>.git
```

The git host ignores the username field, and checks the password only.
Thus any username works. The GitHub form is equivalent, and it makes
the same point clear:

```
https://x-access-token:<token>@git.xhostd.com/<username>/<app>.git
```

Set the remote and push:

```sh
git remote add xhostd "https://<username>:<token>@git.xhostd.com/<username>/<app>.git"
git push xhostd HEAD:master
```

> **Security:** `git remote add` with the token in the URL writes it to
> `.git/config` in plaintext. That one line is a full-scope credential —
> repo write, every channel's database, and the platform API — good until
> the token expires or you revoke it. Use a form that keeps the token
> out of the file: the interactive form or the Bearer header covered in
> the following sections, or a git credential helper
> (`git config --global credential.helper osxkeychain` on macOS,
> `manager` on Windows, `libsecret` on Linux) that stores it in your OS
> keychain. If a repo's `.git/config` is ever exposed, revoke the token
> on the [tokens page](https://console.xhostd.com/tokens) and mint a new
> one. For CI, mint a dedicated token you can revoke on its own.

The `HEAD:master` refspec is deliberate here too. Push under the name you
will deploy, because `deploy(..., ref="master")` binds the channel to that
branch, and a new local repo often uses `main` as its default branch.

Use `git remote set-url xhostd ...` if the remote already exists. `get_app`
and `GET /apps/{app_id}` return the exact `repo_url` of each app.

### Interactive form (no token in the URL)

If you do not want the token in the remote, clone with a plain URL. git
then asks you for the credential:

```sh
git clone https://git.xhostd.com/<username>/<app>.git
```

git asks for a username; type any value. git then asks for a password;
put the token at the **Password** prompt.

### Bearer header (advanced)

git.xhostd.com also accepts the token as a Bearer header. The REST API
and the MCP tools present the token in the same way. Thus one credential
is the same on every surface:

```sh
git config http.extraHeader "Authorization: Bearer <token>"
```

For a person, the Basic-password form is the usual one, because git's
credential helpers use it. An agent uses the Bearer header with
`$XHOSTD_TOKEN`, as the start of this section shows, because the token
value then stays out of every command and file.

## Renew an expired token

When a token expires, a push returns `401`. Create a new token on the
console [/tokens](https://console.xhostd.com/tokens) page and export it
as `XHOSTD_TOKEN`. An agent that registered with its key signs a new
login message;
[Register as an agent](https://docs.xhostd.com/guides/register-as-agent)
shows the command.
If your remote URL holds the old token, update it:

```sh
git remote set-url xhostd "https://<username>:<new-token>@git.xhostd.com/<username>/<app>.git"
```

## A push fails again and again: clear a stale cached credential

Over HTTPS, git keeps the credentials in a **credential helper**. The
helper is the macOS Keychain, `libsecret` on Linux, the Windows
Credential Manager, or a plain `~/.git-credentials` file. This section
applies to the HTTPS path alone, because an SSH push reads no helper.
If a helper holds an old or wrong password for `git.xhostd.com`, git
sends that password again at each push. Your pushes then fail with
`401` or `403`, even after you mint a new token. git does not ask you
again for a credential, because it has one already.

Find which helper is active:

```sh
git config --get credential.helper
```

Then clear the stale entry for `git.xhostd.com`:

- **`store` helper** — edit `~/.git-credentials`, and delete the line
  with `git.xhostd.com` in it.
- **macOS Keychain** — open *Keychain Access*, and remove the
  `git.xhostd.com` internet-password entry. You can also use
  `git credential-osxkeychain erase`.
- **Any helper** — tell git to remove it:

  ```sh
  printf 'protocol=https\nhost=git.xhostd.com\n\n' | git credential reject
  ```

The next push then asks you for a new token, or reads the token from the
URL.

## Connect a GitHub repository with the xhostd GitHub app

The xhostd GitHub app connects a GitHub repository to a project. The app
reads the repositories that you choose on GitHub, and GitHub tells
xhostd about each push. A project that is connected through the app
needs no deploy key and no ping.

### Who can connect

An installation of the app belongs to one GitHub account, a personal
account or an organization:

- On a personal account, the user of that account connects it.
- On an organization, an active owner of the organization connects it.
  A member can only request the install. GitHub sends the request to
  the owners. Until an owner installs the app, the member can connect
  the repository with a deploy key.

### Connect in the console

A person connects the app in the console. No MCP tool and no token can
do it, because the connection needs that person's approval on GitHub.

1. Select **Connect a GitHub repository** on the project's Git page, or
   **Connect GitHub** on the account page under **Git host accounts**.
2. On GitHub, choose the account and the repositories that the app can
   read, and install the app.
3. On GitHub, authorize xhostd to confirm your GitHub account.
4. In the console, choose the repository.
5. Select **Connect** to connect the repository to a project, or
   **Create a project from this repository**.
6. Turn following on for each channel that deploys on push, as
   [Follow the branch](#follow-the-branch) shows.

While the project is connected, you push to the GitHub repository.
`commit_files` and `git push` to xhostd are turned off.

### What an agent sees

`sync_git_from_remote` returns `kind: "installation"` and
`host: "github"` for a repository connected through the app. The
repository needs no setup. Turn following on for each channel that
deploys on push:

```text
update_channel(app_name="shop", channel="prod", follows_branch=True)
```

### Add repositories later

The app reads only the repositories that you chose. To add a
repository, open the repository list in the console and select
**Choose the repositories the xhostd app can read on GitHub**, or select
**Connect GitHub** again. Both open the app's page on GitHub, where you
change the choice.

A repository that is renamed or transferred on GitHub stays connected.
xhostd follows it at the next sync.

### Remove the GitHub account

**Remove** on the account page disconnects the GitHub account from your
xhostd account. Disconnect the account's projects first. When no other
xhostd account uses the GitHub account, xhostd also uninstalls the app
from it. For an organization, that uninstalls the app from the whole
organization.

### Switch a project from a deploy key

1. On the project's Git page, select **Disconnect GitHub**.
2. Connect the repository with the xhostd GitHub app.
3. Turn following on again for each channel, because the disconnect
   turns it off.
4. On GitHub, delete the project's deploy key, and delete the ping: the
   workflow file or the repo webhook.

### Limits

- The app cannot connect a repository whose name starts with `.`, `_`,
  or `-`. For such a repository, **Create a project from this
  repository** creates the project, and the project stays without a
  source.
- The repository filter reads 1,000 repositories at each step. When a
  page shows no match, select **Keep searching** to read the next 1,000.

## Deploy on each GitHub push

This section applies to an app whose source is a GitHub repo connected
with a deploy key, on the Repository source card of the console's Git
page. A repo connected with the xhostd GitHub app needs no ping; read
[Connect a GitHub repository with the xhostd GitHub app](#connect-a-github-repository-with-the-xhostd-github-app).
On such an app you push to GitHub, not to xhostd. A push to GitHub can
deploy on its own, with two steps:

1. The repo tells xhostd that it changed. This is a **ping**: one
   `POST` to the app's notify URL. The ping carries no secret, because
   xhostd trusts nothing in it. xhostd fetches the repo from GitHub
   itself, with the app's read-only deploy key.
2. A channel **follows its branch**. Each ping syncs the mirror. A
   channel that follows its branch then deploys the branch's new head.
   A channel that does not follow keeps its deploys as its own call.

Find the notify URL with `sync_git_from_remote`, which returns it as `notify_url`, or
on the console's Git page. It has the form
`https://api.xhostd.com/webhooks/github/<app-id>`.

### Send the ping with a GitHub Action

Commit this file as `.github/workflows/notify-xhostd.yml`, with your
app's id in the URL:

```yaml
name: Notify xhostd
on: push
jobs:
  notify:
    runs-on: ubuntu-latest
    steps:
      - run: curl -fsS --retry 5 --retry-all-errors -X POST https://api.xhostd.com/webhooks/github/<app-id>
```

The file holds no secret, so an agent with push access can add it alone.
Over HTTPS, a push of a workflow file needs a GitHub token with the
`workflow` scope; an SSH push needs nothing extra.

A branch pings only when its own files carry the workflow. Commit the
file to every branch that should deploy on push.

### Or send the ping with a repo webhook

In GitHub, open the repo's **Settings → Webhooks**, and add a webhook:

- **Payload URL:** the notify URL.
- **Secret:** none.
- **Events:** the push event only.

A webhook covers every branch with no file in the repo. It needs admin
access to the repo, and it uses no GitHub Actions minutes.

### Follow the branch

Turn following on for each channel that should deploy on push:

```text
update_channel(app_name="shop", channel="prod", follows_branch=True)
```

The console has the same switch, **Deploy each GitHub push**, under each
channel on the project's Overview page. Turning it on needs a connected
source repository and a channel bound to a branch. A token that turns it on
needs the `deploy:*` scope too. Turning it off is always allowed.

How a following channel behaves:

- A push to its branch deploys the new head. A push to another branch
  deploys nothing to it.
- After a rewind, the channel keeps the rewound version until its branch
  moves again. The next push to the branch deploys over the rewind.
- A commit that is already queued, building, or serving does not deploy
  twice.
- A failed push deploy, or a push xhostd cannot fetch, sends the app's
  owner a notice on the console and by mail.
- A push made while the app is under maintenance deploys when the
  maintenance ends.
- A commit you force-push away before its deploy fetched it does not
  deploy, and sends no notice.

`sync_git_from_remote` returns `last_ping_at`, the time the last ping arrived. If a
push did not deploy, check that time first: a ping that never arrived
means the workflow file is missing from that branch, or the webhook is
missing.

One notify URL names one app. A repo that feeds two apps needs two
`curl` lines in its workflow, or two webhooks.

## Notes

- Over HTTPS, git.xhostd.com authenticates you with **Basic auth and the
  token as the password**. git's credential helpers use this form.
  git.xhostd.com also accepts an `Authorization: Bearer` header, as
  shown above. The same token works on the REST API (`api.xhostd.com`)
  and on the git host. Over SSH the key alone authenticates you.
- A `401` means that there is no valid token; authenticate again. A
  `403` means that the token is valid, but it gives no access to that
  repo.
- The username in the URL path only locates the repo. The token's role
  on the app decides your access. Thus you can push to every app that
  you own, and to every app where you are a member.
