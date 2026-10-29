# Troubleshooting

Start with the exact project, channel, and time of the problem. Separate a
failed deployment attempt from the revision that is currently serving traffic.

## Contents

- A deployment failed
- The client blocked the action
- The app is slow or unavailable
- The address returns 404 or 502
- The runtime log is unavailable or its status is unknown
- Data or files are missing
- Escalate with useful context

## A deployment failed

Open the project's Activity page in the
[console](https://console.xhostd.com/projects). Select the deployment and read
its build log. Check dependency installation, the launch command, and readiness
before retrying. Preserve the error text without exposing credentials.

A git push alone does not deploy. A queued response is not success. Follow the
[deployment procedure](https://docs.xhostd.com/guides/deployments) through its
terminal state.

## The client blocked the action

Distinguish a client approval prompt from a platform permission error. The
[blocked-deploy guide](https://docs.xhostd.com/guides/client-blocked-deploy)
explains how to identify the source. Do not enable unrelated protected actions
as a workaround.

## The app is slow or unavailable

Check the serving channel, recent changes, and account resource measurements.
The [slow-app guide](https://docs.xhostd.com/guides/diagnose-slowness) walks
through evidence collection. A deployment log explains a build; runtime
metadata describes the process; traffic measurements describe requests.
None substitutes for the others.

Do not assume a missing measurement is zero usage or a failed latest attempt
means the serving app is down. Shared-project members may legitimately lack
the owner's aggregate resource data.

## The address returns 404 or 502

The two codes point at different stages, so read the right log for each.

**404: the channel has no route yet.** Either nobody has deployed the channel
yet, or every deploy so far stopped before the app went live. A deploy gives the
channel its address only after the app passes its health check. Read the deploy
log to find the step that stopped: call `get_logs` with `source="deploy"`, or select the
deployment on the project's Activity page in the console. A failed deploy of a
channel that is already live leaves the previous version serving, so it does
not cause a 404. A 404 that your own app returns is different: the route works,
and the app has no page at that path.

**502: the route exists, but the container does not answer.** The usual causes
are these:

- The app crashed after it went live.
- The app listens on the wrong port, or on `localhost`, not `0.0.0.0`.
  An app that signals readiness with the `$XHOSTD_READY_FILE` file goes live
  without a test of its port. Listen on `0.0.0.0` and on the port in
  `XHOSTD_HTTP_PORT`.
- The app restarted and is still starting.
- The app server closes an idle connection after 1 second or less. The
  platform keeps an idle connection to your app for up to 1 second, so an
  occasional request, usually a POST, gets a 502. Set your server's
  keep-alive timeout to 2 seconds or more.

Read the runtime log to see what the process printed: call `get_logs` with `source="runtime"`,
or open the Runtime status card on the project's Activity page. A background
worker that serves no HTTP returns 502 by design. The
[worker recipe](https://docs.xhostd.com/guides/recipes-worker#the-https-hostname-returns-502)
explains why.

## The runtime log is unavailable or its status is unknown

**The status reads `unknown`.** The reply came from the central log store, and
the platform has no record of how that container ended. This happens when the
host that ran it stopped before it removed the container, or when the container
ran before the platform kept that record. Its output can still be readable:
call `get_logs` with `source="runtime"` and a `command`, such as `tail -n 200 app.log`.

**The header says `lines missing`.** The central log store holds fewer lines
than the container wrote, so your command can miss some of its output.

**The runtime log is unavailable.** A runtime log read can answer with one of
these messages:

- `central log reader busy, retry shortly`: other reads from the central log
  store are running. Retry in a few seconds.
- `runtime log and central store unavailable`: neither the host nor the central
  log store answered. Retry in a minute.
- `runtime log is not available on this host yet`: the host could not serve the
  read. Retry in a minute.

If a message stays for more than a few minutes,
[contact support](mailto:support@xhostd.com) with the project, channel, and
time.

**The console's Logs view shows a notice.** The Logs view on a project's
Activity page can show one of these:

- `The search didn't finish in time.` A page searches for a few seconds. On a
  long time range with a filter, it can stop before it finds a line. Choose
  Load older to keep searching from where it stopped, or choose a shorter range
  or a more specific filter.
- `Logs are unavailable right now.` The central log store did not answer.
  Reload the page in a minute.

## Data or files are missing

Check the target channel and which store the application uses. Review available
recovery points before attempting a restore. See
[Data and recovery](https://docs.xhostd.com/guides/data-and-recovery) for the
distinction between code, database, and file recovery.

## Escalate with useful context

Include the project owner/name, channel, deployment identifier if relevant,
time and timezone, expected result, actual result, and sanitized error text.
Do not include tokens, environment secrets, or customer data. Use console
feedback or [contact support](mailto:support@xhostd.com).
