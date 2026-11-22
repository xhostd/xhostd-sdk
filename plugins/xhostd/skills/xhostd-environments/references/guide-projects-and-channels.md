# Projects and environments

A project owns source code, one or more services, and one or more environments.
An environment is a deployment target, such as production or a preview, with its
own live address. A service is one process of the project, such as a web process
or a worker. Every environment runs the project's services that are turned on in
it. See [Services](https://docs.xhostd.com/guides/services).

## Contents

- Understand the scope
- Create and inspect a project
- Use a preview environment
- Work on a shared project
- Transfer a project
- Earlier names
- Next steps

## Understand the scope

Every project starts with a `prod` environment and one public service, `web`.
Each service that is turned on in an environment uses one slot of the owner's
account-wide allowance. A preview is not a separate billing plan. CPU, memory,
database usage, and file storage are account-scoped resources; adding projects
does not multiply that budget.

An environment of a project that runs code has its own Postgres database. An
environment of a static project does not. Read credentials and hostnames from
the platform-provided environment values; do not derive them yourself. Platform
sites use `xhostd.com`; project addresses use `xhostd.app`.

## Create and inspect a project

[Connect your agent](https://xhostd.com/integrations) to xhostd.
Ask it to create a project with the template your workload needs. Choose a
[working recipe](https://docs.xhostd.com/guides) before writing the build and
launch scripts.

Open [Projects in the console](https://console.xhostd.com/projects). A project's
Overview separates the currently serving environments from the latest deployment
attempt. A failed attempt does not by itself mean the previous version stopped.

## Use a preview environment {#use-a-preview-channel}

Ask your agent to create a separate preview environment and deploy a revision to
it. Review the returned URL before deploying that revision to production. Verify
the environment's variables and data setup; do not assume a preview holds a
current copy of production data.

See the [environment API](https://docs.xhostd.com/api#grp-channels) for the exact
creation and deletion contracts. A new environment and a new code deployment are
different operations.

## Work on a shared project

Check the owner shown next to the project name. The API and MCP tools accept
owner-qualified names where documented; the console keeps that owner context
in links. Membership is not ownership. A member must not interpret an unavailable
owner-wide resource summary as zero usage.

Manage invitations and roles in the project's Access section. See
[Access and security](https://docs.xhostd.com/guides/access-and-security).

## Transfer a project

The owner starts a transfer to a member in the project's Settings, and the
member accepts it in the console's Projects page. The steps, and what each
party can refuse, are in
[Access and security](https://docs.xhostd.com/guides/access-and-security).

**The addresses change at the accept.** Every web address of the project
carries the owner's username, so the accept gives each environment new
addresses with the new owner's username, such as `shop-bob` in place of
`shop-alice`. Each old address keeps working for 30 days, and both
transfer mails state the date. Before that date, update everything that
names an old address: links, webhooks, and sign-in redirect addresses.
After it, the previous owner can use those names for a new project.

**You can rename the project when you accept.** The accept form has an
optional project name field. Leave it empty to keep the name. Enter
another name when your account already has a project, or an address, with
the project's name; the accept tells you when it does. The new name
changes the project's web addresses and its git address. The labels of
its databases and buckets keep the old name.

**External database names change at once.** A database connection that
names the previous owner or the old project name in its database name,
such as `<owner>/<project>/<environment>`, stops working at the accept.
Use the new owner and the new name, or the database's own name, which
never changes.

**Custom domains keep working.** A domain stays verified and keeps
routing. A DNS record that names `cname.<tenant domain>` needs no change.
A record that names an old address keeps working until the date, and
must name `cname.<tenant domain>` after it. The new owner gets a mail
that lists each domain whose record must change, or that the platform
could not read, with the value it must hold.

**A running app sees the new address after its restart.** The accept
restarts each running environment under the new owner. Until that
restart ends, `XHOSTD_HOSTNAME`, `XHOSTD_URL`, and `XHOSTD_DOMAINS` hold
the old values, so an app that checks the Host header refuses the new
address for that time. A visitor who signed in on an old address signs in
again on the new one.

## Earlier names

Earlier versions of xhostd called a project an app, an environment a channel,
and an environment variable an env var. The HTTP API keeps those names in its
paths and fields, such as `/apps/{id}/channels` and `channel_id`. The MCP tools
take the new names: `create_project` was `create_app`, `update_environment` was
`update_channel`, `set_variable` was `set_env`, and the arguments `project` and
`environment` were `app_name` and `channel`. The old tool names, argument names,
and result keys are deprecated and stop working in a later release.

## Next steps

- [Run several services in one project](https://docs.xhostd.com/guides/services).
- [Push source with git](https://docs.xhostd.com/guides/git).
- [Deploy and inspect a change](https://docs.xhostd.com/guides/deployments).
- [Understand recovery](https://docs.xhostd.com/guides/data-and-recovery).
