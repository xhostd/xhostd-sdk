---
name: xhostd-environments
description: >-
  Manages the environments and services of an xhostd project. Starts when the
  user wants a preview or staging environment for a branch, a branch binding or
  deploy-on-push changed, an environment deleted, a second process such as a
  worker or an API added, or a service turned on, turned off, renamed, deleted,
  or restarted. Holds environments, branch bindings, follows_branch, services,
  private service names, and the environment URL formats.
---

# xhostd environments and services

An environment is a deployment target with its own URL, such as `prod` or a preview. A service is
one process of the project, such as a web process or a worker; every environment runs the services
that are turned on in it.

## Rules for every xhostd task

1. **Write only on request** (create, delete, push, deploy, restore, expose, change); ask if unsure.
2. **No credential in a plain variable**: store it with `set_secret`, never `set_variable`.
3. **No token value in a command**, URL, file you might commit, or chat: read it from a variable.
4. **No plan advice** or purchase link unasked. On `plan_limit_exceeded`, state the limit; no retry.

## Environments

Every project has one `prod` environment, created automatically; the first deploy with `ref`, or
`create_project`'s `branch`, binds it to a branch. To change the branch an environment deploys
without deploying, call **`update_environment`** with `project`, `environment`, and `branch`. For a
preview or staging environment call **`create_environment`** with `project`, `name`, and
`git_ref_binding` (`branch:<name>`, one explicit environment per branch), then `deploy` to it.
`delete_environment` removes an environment other than `prod`, with its database, and cannot be
undone.

On a project with a connected source repository, an environment can **follow its branch**: call
**`update_environment`** with `follows_branch=True` (it needs the `deploy:*` scope), then each push
to that branch on the source repository deploys the new head, and that environment's deploy is no
longer its own call. A push deploys only the services whose directory changed. The `kind` that
`sync_git_from_remote` returns says what setup the push needs: an `installation` source needs
none, and a `deploy_key` source must ping the `notify_url` that `sync_git_from_remote` returns,
whose description links the two ways to send the ping. A push to another branch never redeploys a
following environment, so a rewind holds until its own branch moves.

URL format:
- Prod: `https://<project>-<owner>.xhostd.app`
- Other environments: `https://<environment>-<project>-<owner>.xhostd.app`

`<owner>` is the user's xhostd username. Read the exact `hostname` from the environment object that
`create_project`, `create_environment`, or `get_project` returns, and never construct it. Each
environment also answers on its full address, which names the service; `addresses` lists every
address.

## Services

A project starts with one public service, `web`, which holds the addresses. **`create_service`**
adds a service that is not public, such as a worker, with its own `command` and `directory`; it
starts at each environment's next deploy. **`update_service`** renames a service or changes its
`command`, `directory`, `memory_mb`, or `rollout`; with `environment` and `enabled` it turns the
service on or off in one environment. A turn-off stops the service at once and frees its slot; the
environment keeps its database and backups. **`delete_service`** removes a service that is not
public. **`restart`** starts services again with their current image and no build, which applies a
changed setting or variable. Restarting a service, turning one off, and deleting an environment are
write operations: run them only when the user asks for that operation on that target.

Services of one environment reach each other by service name, such as `http://api:8000`, so a
service listens on `0.0.0.0`. A service that is not public is ready when it creates
`$XHOSTD_READY_FILE`, or after 10 seconds of running with no restart; create the file when the
service needs an exact signal. Each service that is turned on in an environment uses one slot of the
plan. Relay a write's `notice`. `references/guide-services.md` holds the rules, the limits, and
worked examples.

## References

- `references/guide-projects-and-channels.md` — Projects & environments: Project scope, environment
  isolation, and shared account capacity.
- `references/guide-services.md` — Services: a web process, a worker, and an API as services of one
  project; private names, the `rollout` setting, the push filter, and the limits.
