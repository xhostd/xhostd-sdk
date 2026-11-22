# Services

A service is one process of a project, such as a web process or a worker. Every
project starts with one service, `web`, which is public: it holds the project's
addresses and answers HTTP. You can add services that are not public, such as a
worker that processes jobs, or an API that only the web process calls. Every
service of a project builds from the same repository, and every environment of
the project runs every service that is turned on in it.

## Contents

- What a service holds
- Run a web process and a worker
- Run a front end and an API from two directories
- Static services
- Call another service by its name
- Choose a rollout
- Memory
- Variables
- Turn a service off in one environment
- Deploys that you do not start
- Rename a service
- Change a service's directory
- Limits
- Commands in images

## What a service holds

A service has these settings:

| Setting | Meaning | Default |
|---|---|---|
| `name` | 1 to 16 lowercase letters, digits, or hyphens. `prod`, `localhost`, a name that starts with `xhost-`, and a 12-character hexadecimal name are reserved. A service and an environment of one project cannot share a name. | None; you choose it |
| `command` | The command that starts the process, on one line. | `./launch.sh` for the `app` template, the image's own command for `docker` |
| `directory` | The directory of the repository that the service builds from. | The repository root |
| `template` | `static`, `app`, or `docker`. It never changes after create. | The project's template |
| `public` | Whether the service holds the addresses. A project has one public service for now. It never changes after create. | `false` |
| `memory_mb` | A memory limit of its own, from 64 MB to your plan's memory limit. | No limit of its own |
| `rollout` | `replace` or `overlap`. See [Choose a rollout](#choose-a-rollout). | `overlap` for the public service, `replace` for any other |

Each service that is turned on in an environment uses one slot of your plan.
A create that needs more slots than your plan leaves free is refused, and the
refusal states the count.

## Run a web process and a worker

This example runs a web process and a worker from one repository. Both share
the environment's database. The web process inserts a job per request, and the
worker processes the jobs.

1. Commit `install.sh`, `launch.sh`, `web.py`, and `worker.py`. `launch.sh`
   starts the web process: `exec python web.py`. The worker creates the file
   that `$XHOSTD_READY_FILE` names after its first query, and it handles
   `SIGTERM`.
2. Add the worker:

   ```text
   create_service(project="shop", name="worker", command="python worker.py")
   → {"service": {...}, "notice": "The service starts at each environment's next deploy."}
   ```

   A create starts nothing. `get_project(project="shop")` lists `worker` as
   `pending` in each environment until its next deploy.
3. Deploy the commit:

   ```text
   deploy(project="shop", environment="prod", sha="<the sha of step 1>")
   ```

   The deploy builds the repository once, starts `web`, and then starts
   `worker`. Each service records its own result. A deploy that names no
   `services` deploys every service that is turned on in the environment.

To deploy one service, pass `services=["worker"]`. To start a service again
with its current image, call `restart(project, environment, services=["worker"])`.
To read one service's logs, pass `service="worker"` to `get_logs` with
`source="runtime"`. `get_health(project, environment)` reports each service.

## Run a front end and an API from two directories

This example builds two services from two directories of one repository. The
web process serves the front end and forwards `/api` to the API service.

1. Before the first deploy, point the public service at its directory:

   ```text
   update_service(project="duo", service="web", directory="web")
   ```

2. Add the API:

   ```text
   create_service(project="duo", name="api", directory="api", command="python api.py")
   ```

3. Commit `web/launch.sh` with a server that forwards `GET /api` to
   `http://api:8000`, and `api/api.py` with a server on port 8000 of every
   address of its container.
4. Deploy the commit with `deploy(project="duo", environment="prod", sha="<sha>")`.
   The deploy builds each directory once, and each image tag names its
   directory.

A command runs in the service's directory. Two services that share a directory
and are not `static` must use the same template.

## Static services

A `static` service is served from the repository root, so it takes no
directory. A project that holds a `static` service keeps every service at the
repository root.

A project whose public service is `static` holds `static` services only: a
static service serves every file of the repository, so a process beside it
would have its source published. To run a worker beside a site, create a
project with the `app` template whose public service runs a process that serves
the site's files, and add the worker there.

## Call another service by its name

In a project with two or more services, each service reaches another by the
service's name, such as `http://api:8000`. A service must listen on every
address of its container, such as `0.0.0.0`, to answer such a call. What the
platform promises:

- A name resolves only to an instance that passed its readiness check once.
- A name is not a health check: an instance that crashes and restarts keeps
  its name.
- The platform promises no load balancing, no answer during a `replace`
  deploy, and no single version during an `overlap` deploy. A caller must
  retry.
- Every service of an environment can call every other.
- Names exist only in a project with two or more services.
- A rename applies at the environment's next deploy, rewind, or restart, which
  can be a restart that the platform runs, with no alias.
  [Rename a service](#rename-a-service) states what else changes.
- No name crosses environments or projects.

## Choose a rollout

The `rollout` setting chooses how a deploy replaces a service's running
instance.

| Aspect | Public service (`overlap`) | Other service (`replace`) |
|---|---|---|
| Replacement | The new instance passes its check before the old one stops | The old instance stops before the new one starts, so no job runs twice. The gap is about 11 seconds. |
| Readiness | The port answers, or the ready file exists | The ready file exists, or 10 seconds pass with no restart |
| Stop grace | 5 seconds | 30 seconds |

You can set `overlap` on a worker to remove the gap. Two versions then run
together for some seconds, the memory use doubles during the swap, and the
protection that no job runs twice is gone.

A service that is not public is ready when it creates the file that
`$XHOSTD_READY_FILE` names, or after 10 seconds of running with no restart;
create the file when the process can work and the service needs an exact
signal. A worker with `overlap` signals the same way. A service that is not
public receives no `XHOSTD_HTTP_PORT`, no `XHOSTD_URL`, and no other address
variable.

## Memory

Every service of your account shares your plan's memory. On a plan of 128 MB,
a second process might not fit. When memory runs out, the platform stops a
service that is not public first. Set `memory_mb` on a service to give it a
limit of its own.

## Variables

Every service of an environment receives the project's variables and the
environment's variables. Each container also receives `XHOSTD_PROJECT`,
`XHOSTD_ENVIRONMENT`, and `XHOSTD_SERVICE`, which hold the project name, the
environment name, and the service name.

## Turn a service off in one environment

```text
update_service(project="shop", service="worker", environment="preview", enabled=False)
```

A turn-off stops the service in that environment at once and frees its slot.
The environment keeps its database, its files, and their backups. To turn the
service on again, pass `enabled=True`; it starts at the environment's next
deploy or restart. The public service stays on in every environment and cannot
be deleted, because it holds the addresses.

`delete_service(project, service)` removes a service that is not public from
every environment and stops its containers.

## Deploys that you do not start

- **A push to a followed branch** deploys only the services whose directory
  changed. A service at the repository root always deploys. A change to a
  file outside a service's directory does not deploy that service. A service
  whose directory setting changed since its last build deploys too.
- **A platform redeploy** keeps each service on the commit it runs.
- **`rewind_deploy(project, environment)`** returns each service to its newest
  earlier successful deploy of another commit. Pass `service` to rewind one
  service.

## Rename a service

Rename a service with `update_service` and `name`, the console's Services
card, or `xhostctl service update <project> <service> --name <new>`. A rename
refuses nothing because a deploy is queued or running. A deploy in progress
finishes with the names it read when it started.

| What | When it changes |
|---|---|
| The service's name | At once |
| Each environment's full address of a public service | Within a few seconds. The old address keeps working until the date that the result states, 30 days later, and another name can take it after that date. |
| The environment's short address | Never |
| The name other services of the environment call | At the environment's next deploy or restart, which can be a restart that the platform runs |
| `XHOSTD_SERVICE`, and the new address in `XHOSTD_DOMAINS` | At the service's next deploy or restart |
| Custom domains and public TCP endpoints | Never. Each keeps pointing at the service. |

Change the callers before the environment's next deploy or restart: the old
name stops resolving at the same moment that the new one starts. An app that
checks the `Host` header against `XHOSTD_DOMAINS` refuses the new address
until that deploy or restart, and the old address and the short address keep
answering. If a custom domain's DNS record names an old address, point the
record at the generic target that the console shows before the old address's
date. An environment can hold 5 old addresses that still work; a rename that
would add a sixth is refused with the date the oldest one stops.

## Change a service's directory

You can change `directory` at any time. The notice states: "The new directory
applies at the service's next deploy." Until then the service keeps the image
built from its previous directory, and a restart, a rewind, and a platform
redeploy do not apply the change.

A directory stays with the template of the service whose image was built from
it, until that service is deleted. A service of another template that names
such a directory is refused: "service '`<b>`' has built from the directory
'`<dir>`' with the template '`<t>`', so service '`<a>`' cannot use it with
another template. Use another directory, or delete '`<b>`'."

## Limits

- A project has one public service.
- The public service stays on in every environment and cannot be deleted.
- A project with a `static` service keeps every service at the repository root,
  and a project whose public service is `static` holds `static` services only.
- A service change is refused while the platform runs maintenance on your
  account. The refusal says so; try again later. A deploy never
  blocks a service change: a turn-off and a delete act at once, and a deploy
  in progress skips that service.

## Commands in images

- On the `docker` template, the platform passes `command` as arguments to the
  image's entrypoint. An image with a shell-form `ENTRYPOINT`, such as
  `ENTRYPOINT python app.py`, ignores them. Use the exec form, such as
  `ENTRYPOINT ["python"]`, or no entrypoint.
- On the `app` template, an image built before the platform supported
  commands refuses a command. Deploy the service again to build a new image.
