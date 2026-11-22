---
name: xhostd-data
description: >-
  Works with an xhostd project's data. Starts when the user wants the project's
  Postgres database added, connected to, or reached from outside, a snapshot
  listed, restored, or downloaded, S3-compatible file storage used, a database
  or bucket renamed, or the project's data exported. Holds DATABASE_URL and the
  pooler, the Postgres password, snapshots and the restore modes, object
  storage, and exports.
---

# xhostd data: Postgres, snapshots, files, and exports

A restore is a write operation: start one only when the user asks to restore that environment,
after the user confirms the environment, the checkpoint, and the mode. On `prod`, a restore is a
protected action: an agent credential gets 403 `protected_action` until the project owner turns
agent access on in the console.

## Rules for every xhostd task

1. **Write only on request** (create, delete, push, deploy, restore, expose, change); ask if unsure.
2. **No credential in a plain variable**: store it with `set_secret`, never `set_variable`.
3. **No token value in a command**, URL, file you might commit, or chat: read it from a variable.
4. **No plan advice** or purchase link unasked. On `plan_limit_exceeded`, state the limit; no retry.

## Postgres

Every environment of a project that runs code automatically gets `DATABASE_URL` pointing at the
environment's own Postgres database — read it from `process.env` (or equivalent); don't ask the user
for a connection string. There is no enable step. A static project has no database, so it gets no
`DATABASE_URL`.

`DATABASE_URL` can go through a transaction pooler, so run migrations and listeners over
`DATABASE_URL_DIRECT`, which always reaches the server itself.
The Postgres recipe, <https://docs.xhostd.com/guides/recipes-postgres.md>, holds the raw text of
what else needs `DATABASE_URL_DIRECT`, and the migration pattern.

Your `$XHOSTD_TOKEN` token is also your **Postgres password** when external database access is
enabled in the console. Connect as the user `<username>` to the host `db.xhostd.com` on port `5432`,
with `sslmode=require`. The database `<db>` is the project name for `prod`, else
`<environment>-<project>`. `psql` reads the token from the `PGPASSWORD` environment variable, so the
token stays out of the connection string. A scoped token with `scopes=["db:*"]`, minted by the
request `get_credential_request` describes, works the same way.
`references/guide-data-and-recovery.md` gives the exact command.

`rename_database` and `rename_bucket` set the label that names an environment's database or bucket
in listings; they change no connection string.

## Snapshots

An environment has two kinds of snapshot, and the `kind` field names the kind. Every non-static
deploy first takes a `pre_deploy` snapshot of Postgres, and xhostd keeps the newest
`deploy_snapshot_keep` of them with no age cap. A `nightly` snapshot is a routine copy, deleted
after the plan's `snapshot_retention_days`; `get_stats(view="overview")` reports both figures.
xhostd always keeps an environment's newest `nightly` snapshot, however old it is, and an
environment with no database gets no `nightly` snapshot.

`list_snapshots` lists both kinds newest-first. `restore_snapshot` with `part="database"` rolls the
environment's database back to a snapshot, and `part="files"` rolls its object storage back to a
checkpoint whose `aligned_blob` flag is true. The restore returns at once and runs in the
background. A restore can take up to an hour; for databases under 10 GB it typically takes minutes.
Tell the user that before you start, then poll `get_restore_status` and relay its `activity`
sentence. Agree the `during_restore` mode with the user (`read_only` is the default); the tool's
description states what each mode does to the app.

`get_snapshot_download_link` returns the download link for a snapshot's database, files, or code,
and mints nothing: fetch `download_url` with your own token, or give the person `console_url`.
`references/guide-data-and-recovery.md` gives the exact command.

## Object storage (S3-compatible)

Each environment gets object storage automatically, like the database of a project that runs code,
for unstructured blobs: uploads, generated assets, exports. Deploys inject `S3_ENDPOINT`,
`S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `S3_REGION` — point any S3 SDK at those
env vars, and do not construct them. The blob recipe,
<https://docs.xhostd.com/guides/recipes-blob.md>, holds the raw text of uploads and serving.

From outside the project, `get_credential_request` with `purpose="object_storage"` returns the
request that answers with the endpoint, bucket, and key pair: run it with your own
`$XHOSTD_TOKEN` into a file only you can read; the tool result never holds the key pair.
`get_stats` with `view="object_storage"` reports the bytes used. A client outside xhostd also needs
the bucket's external-access toggle, which a person turns on in the console; no tool does.

## Exports

`export_data` queues a portable takeout of an environment or a whole project, which standard tools
reload with no xhostd. Poll `get_export_status`; when the export is ready it returns the archive's
`download_url` and `console_url`. Fetch them as you fetch a snapshot download.

## References

- `references/guide-data-and-recovery.md` — Data & recovery: Postgres, files, recovery points,
  exports, restore safeguards, the `psql` command, and the download command.
