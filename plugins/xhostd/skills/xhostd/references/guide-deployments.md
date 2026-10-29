# Deployments

Deploy a specific revision to a specific channel, then inspect the outcome.
Sending source to the repository and making it live are separate operations.

## Push, then deploy

Use [git](https://docs.xhostd.com/guides/git) to send code to the project's
repository. A push stores the commit; it does not deploy it. Use `commit_files`
only when git is unavailable or neither git transport can push, as described in
[Ship without git](https://docs.xhostd.com/guides/recipes-commit-files).

Call the [deploy tool](https://docs.xhostd.com/mcp-tools#deploy) with the app,
channel, and commit. Pass a full SHA when you need the same immutable target
across review and execution. A branch reference resolves at deploy time.

## Follow the operation

The deployment call returns queued work, not proof of success. Read
[get_logs](https://docs.xhostd.com/mcp-tools#get_logs) with `source="deploy"` and inspect its
status until it is terminal. Build output and runtime output answer different
questions. A missing build log does not establish that the running app is healthy.

The project's Activity page in the [console](https://console.xhostd.com/projects)
separates deployment history, build-log windows, runtime metadata, log lines, and
audit events. Use Refresh to read additional build output. The Logs view shows
the lines a channel's containers wrote, newest first, from the central log store.
Filter them by container, time range, and text, and choose Load older to read
further back. The console does not run shell commands on a log and does not
stream new lines as they arrive; reload the page to see them.

## Read the runtime log after a move or a host failure

[get_logs](https://docs.xhostd.com/mcp-tools#get_logs) with `source="runtime"` reads a
container's output from the host that ran it. The platform also keeps each
container's output in a central log store, for the retention period that the
account's plan sets, so the output survives when that host fails and when
your project moves to another host.

When the host that ran a container does not answer, or no longer holds it, the
reply comes from the central store, and its header says
`source: central log store`. A `command` runs on those lines the same way.

A central read keeps the newest 200,000 lines or 64 MiB of a container's
output. When it leaves out older lines, the header says
`older lines omitted`. Your command sees only those newest lines. When the
central store holds fewer lines than the container wrote, the header says
`lines missing`.

## Deploy an existing revision from the console

Open the project Overview and choose its deployment action. Select the channel
and enter the full 40-character commit SHA. Review the project, owner, channel,
and current serving revision before confirming.

The review expires after ten minutes and is tied to your session and target.
If the serving revision changes, review again. Permission is checked again at
execution; an old review does not retain access you no longer have.

## Recover code without confusing it with data

The [rewind_channel tool](https://docs.xhostd.com/mcp-tools#rewind_channel)
uses a retained previous successful image where supported. Deploying an older
SHA rebuilds that revision. Neither operation is a database or file restore.

> Check the application's schema expectations before running older code against
> current data. A deployment review does not freeze environment values or undo
> data changes.

For database and file recovery, use
[Data and recovery](https://docs.xhostd.com/guides/data-and-recovery). For failures,
start with [Troubleshooting](https://docs.xhostd.com/guides/troubleshooting).
