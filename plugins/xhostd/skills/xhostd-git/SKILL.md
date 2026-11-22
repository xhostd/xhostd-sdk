---
name: xhostd-git
description: >-
  Pushes code to an xhostd project's git repository. Starts when code goes to an
  xhostd repository with git, when a git remote, an SSH key, or a push to
  git.xhostd.com fails, or when git over HTTPS needs an xhostd token. Holds the
  SSH push with a key of the connected account, the HTTPS push with the token in
  a header, the rules for keys and tokens, the scoped, short-lived token, and a
  project's connected source repository.
---

# Push code to xhostd with git

Every xhostd project owns a git repo, and a push to it puts code onto the project, for every project
whatever its size, from the first commit on. **A push stores code; it does not deploy.** Start the
build with `deploy` afterwards. Push only when the user asked for the change. Full guide:
`references/guide-git.md` (<https://docs.xhostd.com/guides/git>).

**Pick the path from what the machine can do, before you push — never after a failure:**

- **No shell** — a client with no shell access has no git. Use `commit_files`, then `deploy` the
  `sha` it returns. `sync_git_from_remote` has no part in this flow: it refreshes the mirror of a
  project whose code comes from a connected source repository.
- **A shell** — push over **SSH**, steps S1–S6 below.
- **Outbound port 22 blocked, or the SSH push fails** — push over **HTTPS** with `$XHOSTD_TOKEN`,
  steps H1–H5 below.
- **Neither transport can push** — no usable SSH key, and no `$XHOSTD_TOKEN` in the environment. Use
  `commit_files`, as with no shell. Never ask the user to paste a token.

## SSH — with a shell

S1. Get the project's `repo_url` via `get_project` (`project`). It looks like
`https://git.xhostd.com/<username>/<project>.git` and carries the `<username>` and `<project>` the
SSH remote needs. **An SSH push needs no token.**

S2. Find a key that the connected account holds: read the fingerprint of every existing
`~/.ssh/xhost_ed25519*.pub` with `ssh-keygen -lf <file>`, and compare each with the `fingerprint`
values that `list_ssh_keys` returns. A match proves that the key pushes for this account, so push
with the key whose `.pub` file `list_ssh_keys` lists (S6). A file name proves nothing about the
account; only a match does.

S3. If none is listed, register the default key. If `~/.ssh/xhost_ed25519` is absent, make it in a
subprocess — `ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519` — so the private half goes
straight to disk and never enters a tool call. Call **`register_ssh_key`** with the content of
`~/.ssh/xhost_ed25519.pub` and a `label` that names the machine, then push with that key (S6).

S4. A fingerprint is unique platform-wide, so a `409` in S3 means that the default key belongs to a
different xhostd account on this machine. Make `~/.ssh/xhost_ed25519_<username>` in a subprocess if
it is absent (`ssh-keygen -t ed25519 -N "" -f ~/.ssh/xhost_ed25519_<username>`), register the
content of its `.pub` file with `register_ssh_key`, and push with it (S6).

S5. If that registration also answers `409`, this machine holds no usable SSH key for this account.
Push over HTTPS (H1–H5) when `$XHOSTD_TOKEN` is set; otherwise use `commit_files`. Never make a key
under any other name.

S6. `git remote add xhostd-ssh "git@git.xhostd.com:<username>/<project>.git"`, then push
`HEAD:master` to that remote with the key that S2–S4 chose as the only identity ssh offers: git's
`GIT_SSH_COMMAND` names the key, and `-o IdentitiesOnly=yes` stops ssh from offering another key it
finds first. Then trigger the build with **`deploy`**. `references/guide-git.md` gives the exact
command.

`list_ssh_keys` returns the account's keys (metadata only) and `delete_ssh_key` revokes one by
`key_id`.

## HTTPS — when port 22 is blocked or the SSH push fails

H1. Check that `XHOSTD_TOKEN` is set in the environment (`test -n "$XHOSTD_TOKEN"`), never by
printing it. It holds your own `xh_` token: one general credential that is your git password, your
Postgres password, your download credential, and your platform API bearer at once. If it is unset
and SSH cannot push, use `commit_files`; never ask the user to paste a token.

H2. Get the project's `repo_url` via `get_project` (`project`). It looks like
`https://git.xhostd.com/<username>/<project>.git`.

H3. Add the remote with no token in it:
   ```
   git remote add xhostd "https://git.xhostd.com/<username>/<project>.git"
   ```
   (or `git remote set-url xhostd ...` if it already exists).

H4. Push `HEAD:master` (or `HEAD:<your-branch>`) to the `xhostd` remote, and send `$XHOSTD_TOKEN` as
an `Authorization: Bearer` header that git reads from three `GIT_CONFIG_*` variables in its
environment, under a key scoped to `git.xhostd.com` (git 2.31 or later). The token then never
appears on a process's argv, in `.git/config`, or in the transcript; never write the token value
itself into a command. `references/guide-git.md` gives the exact command.

H5. Trigger the build with **`deploy`**. Pass `ref: "master"` (or the branch name) so xhostd
resolves to HEAD; the `ref` binds the environment to that branch. Or pass an explicit `sha`, which
binds nothing.

Both transports reach the same repo. Push under the branch name you then deploy with `ref`:
`HEAD:master` pushes the current branch as `master`, whatever a fresh `git init` named it locally,
and `deploy(..., ref="master")` deploys it and binds the environment to `master`.

**A scoped, short-lived token.** Instead of `$XHOSTD_TOKEN` itself, you can mint a scoped,
short-lived token from it, which suits an unattended job or a token you hand to another process.
`get_credential_request` never returns a credential: it describes the `POST /credentials` request,
which you run into a variable of its own, such as `XHOSTD_GIT_TOKEN`, in the same shell command as
the push. `references/guide-git.md` gives the exact command.

## Keys and tokens

- **Keep every key at `~/.ssh/xhost_ed25519` or `~/.ssh/xhost_ed25519_<username>`.** Never put a
  key in the project directory, and never add a per-project or per-tool suffix. Those paths are in
  `$HOME`, so every session and project on the machine reuses one key per account.
- **Never make a second key for the same account**, and never make one to clear a `409` for a key
  that `list_ssh_keys` lists for this account. Each new key registers on the account and notifies
  the user.
- **Keep a token out of every repo, every transcript, and every env file you might commit.** Never
  put a token in a URL, and never write its value into a command.
- **A token expires.** After expiry, ask the user for a fresh token from the console /tokens page,
  or, for a registered agent, renew the token with the key, as
  [Register as an agent](https://docs.xhostd.com/guides/register-as-agent) describes.

## A connected source repository

A project whose code comes from a repository connected in the console refuses `commit_files`: a
push to that repository reaches the project. `sync_git_from_remote` refreshes the project's mirror
without a deploy and reports the sync status; each deploy fetches first. `xhostd-environments`
covers an environment that deploys each push to its branch.

## References

- `references/guide-git.md` — Push code with git: the unified credential, the HTTPS push with
  `$XHOSTD_TOKEN` as a header, the `HEAD:master` refspec, and connected repositories.
