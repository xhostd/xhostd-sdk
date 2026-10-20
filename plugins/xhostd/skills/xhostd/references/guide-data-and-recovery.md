# Data and recovery

Understand which data your project uses and which recovery point you are about
to apply. Code, Postgres, and files are distinct parts of the application.

## Choose the right store

Code-running templates receive Postgres connection settings. Static sites do
not get a database. Follow the
[Postgres recipe](https://docs.xhostd.com/guides/recipes-postgres) for schema
migrations and database access.

Use S3-compatible object storage for uploads and durable app files. Follow the
[file upload recipe](https://docs.xhostd.com/guides/recipes-blob). Do not treat a
container's writable layer as a backup of your data.

## Read usage at the right scope

The console's Usage area shows account resource measurements. Database usage
has a soft allowance; file storage has an enforced limit. Measurements can lag
recent writes. A member of a shared project cannot read the owner's account-wide
resource totals. An unavailable measurement is not zero.

## Review recovery points

Open the project's Data & recovery section. Check the channel and timestamp,
whether code was recorded, and whether the file checkpoint is aligned with
the database checkpoint. A checkpoint without an aligned file marker does not
represent an atomic restore of the whole app.

Nightly snapshot retention differs by plan. Pre-deploy recovery points use a
separate count policy. Read the current values from `get_stats(view="overview")` and the
[backup policy](https://xhostd.com/faq#backups) before relying on a particular
recovery window.

## Restore with an exact target

A database restore runs in the background. A restore can take up to an hour;
for databases under 10 GB it typically takes minutes. Follow it on the
project's Database page, or with `get_restore_status`.

Choose what your app does while the restore runs:

- **Read-only**, the default: your app stays online and reads its data, and
  every database write fails until the restore ends.
- **Maintenance**: your app's addresses show a maintenance page, and every
  database write fails until the restore ends. xhostd's own page answers with
  HTTP 503, so an uptime monitor reports your app as down during the restore.
- **Live**: your app keeps writing, and the restore discards those writes when
  it switches to the restored database.

In every mode, a write your app makes while the restore still waits to start
is discarded at that switch.

To show your own maintenance page, with its own pages, images, and styles,
deploy a static site first. Then choose it as the page in the console, or pass
its app and channel to `restore_channel_snapshot` as `maintenance_app` and
`maintenance_channel` with `during_restore="maintenance"`. Your site answers
with its own status codes: 200 for its pages, and 404 for a path it lacks.
Don't redeploy your page channel while a restore runs: for up to a minute
after the redeploy, requests to your app's addresses fail.

Database and file restores retain their own confirmation and permission rules.
Review the project, channel, source checkpoint, and impact on current data.
Copying a restore command is not approval to execute it.

After a restore, verify the application's behavior and its data. Deploying old
code alone does not restore the database. See
[Deployments](https://docs.xhostd.com/guides/deployments).

## Export and download

Use the project's Exports page from Data & recovery for supported takeout flows.
Checkpoint downloads and exports can have different availability and size
restrictions. A disabled feature, unavailable archive, or incomplete export must
not be treated as a successful backup.

Read the [export reference](https://docs.xhostd.com/mcp-tools#exports) for the
operation's scope, status, and result. Keep downloaded data and credentials out
of public bug reports.
