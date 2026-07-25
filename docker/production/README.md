# Production Docker stack

This directory holds a deployable stack for Frappe CRM. It is **not** the same
thing as `../docker-compose.yml`, which is a development bench.

| | `../docker-compose.yml` | this stack |
| --- | --- | --- |
| Image | `frappe/bench:latest` | built from `./Dockerfile` |
| Startup | Runs `bench init`, downloads and builds Frappe | Starts the built app |
| Time to first response | ~10-30 minutes | seconds (plus one-time site creation) |
| Web server | `bench start` (dev server) | gunicorn behind nginx |
| Needs a host bind mount | Yes (`.:/workspace`) | No |
| Deploys your local code | No, pulls `crm` from GitHub `main` | Yes, via `CRM_REPO`/`CRM_BRANCH` build args |

The dev stack cannot pass a platform health check, for the reasons in the first
two rows: its entrypoint script lives on the host and is bind-mounted in, and
even when that works it does not listen on a port until the build finishes.

## Why the image is built rather than pulled

The obvious approach — pull `ghcr.io/frappe/crm:stable` — does not work. **Those
published images do not contain the crm app.** Verified against the raw images:

```
$ docker run --rm --entrypoint ls ghcr.io/frappe/crm:stable -1 \
    /home/frappe/frappe-bench/apps
frappe
```

`:main` is identical. `bench new-site --install-app crm` against either dies with
`Could not find app "crm": No module named 'crm'`.

The cause is upstream. `.github/workflows/builds.yml` passes the app list as an
`APPS_JSON_BASE64` **build arg**, but frappe_docker's
`images/layered/Containerfile` now reads it from a BuildKit **secret**
(`--mount=type=secret,id=apps_json`). The build arg is silently ignored, so
`bench init` runs with no `--apps_path` and produces a frappe-only image.

`./Dockerfile` layers the app onto the published image, which already carries a
working bench, `entrypoint.sh` and `nginx-entrypoint.sh`. If upstream fixes the
workflow, this layer can be dropped in favour of a plain `image:` line.

## Deploying

```bash
cp .env.example .env
$EDITOR .env          # set DB_ROOT_PASSWORD, ADMIN_PASSWORD, SITE_NAME
docker compose up -d --build
```

The first build compiles CRM's frontend assets and takes several minutes. To
deploy your own fork instead of upstream `main`, set the `CRM_REPO` and
`CRM_BRANCH` build args in `docker-compose.yml`.

Point the platform at `docker/production/docker-compose.yml`, and mark
**`frontend`** as the primary service — it is the only one publishing a port.

### Deploying from a platform that clones this repo

`.env` is gitignored, so it does **not** exist in a fresh clone. Every variable
therefore falls back to a default and the file parses without it — including
`DB_ROOT_PASSWORD` and `ADMIN_PASSWORD`, which both default to `admin`.

**Set them as environment variables in the platform's own settings UI before
putting this on a public address.** Do not rely on the defaults.

Defaults are used rather than Compose's `${VAR:?error}` required-variable syntax
on purpose: `:?` aborts interpolation of the *entire* file when the variable is
unset, so the platform sees zero services and zero ports rather than a helpful
error.

## Health checks

`frontend` has a Compose-level health check against **`/crm`** with a 300s
`start_period`. On the very first deploy the `create-site` job has to build the
database and install the app before that path answers, which takes a few minutes.

It deliberately does *not* probe `/api/method/ping`. Frappe answers `ping` with
`200 {"message":"pong"}` as long as *any* site resolves — including a site where
the CRM app failed to install. A stack in exactly that state reports `healthy` on
`ping` while `/crm` returns 404, so the platform would call a dead deploy green.

The check treats **403 as healthy**. `/crm` requires a login, so an anonymous
probe gets 403 when CRM is installed and 404 when it is not — accepting 200/403
while rejecting 404 is precisely what separates a working deploy from a broken
one. If you point the platform's own check at a URL, use `/crm` and configure it
to accept 403, or use `/api/method/ping` only as a liveness (not readiness)
probe.

## Startup ordering

`backend`, `websocket`, `scheduler` and both queue workers wait on
`create-site` via `condition: service_completed_successfully`.

This is not cosmetic. When `backend` was allowed to start alongside `create-site`
it served requests before the site existed, cached an empty installed-apps list
in Redis, and then returned **HTTP 500 on every page** — `AttributeError:
'ErrorPage' object has no attribute 'app_path'`, wrapping
`AppNotInstalledError: App frappe is not installed` — indefinitely, even after
the site was created successfully. Only `bench clear-cache` plus a `backend`
restart recovered it. Gating on `create-site` prevents the bad cache entry from
ever being written.

A consequence worth knowing: if `create-site` fails, the dependent services never
start, so nothing binds the port and the platform reports a failed deploy. That
is intentional — the alternative is a green health check in front of a dead app.

**Set the platform's health check grace period to at least 5 minutes.** A probe
that starts failing the container after 30-60s will kill the stack mid-site
creation, and because the half-created site persists in the `sites` volume, the
retry can come up in a broken state. If that happens, clear the volumes and
redeploy:

```bash
docker compose down -v && docker compose up -d
```

Subsequent deploys are fast — `create-site` sees the existing site and exits.

## Port

`frontend` publishes **`8052:8080`** as a literal in `docker-compose.yml`. Inside
the container nginx always listens on 8080; change the public port by editing the
left-hand number.

Do not convert that mapping back into a variable such as
`"${CRM_HTTP_PORT:-8052}:8080"`. Deploy platforms commonly scan the compose file
with their own YAML parser to find the public entrypoint, and those parsers do
not expand `${VAR:-default}` — the templated value reads as *no host port
published* and the deploy is rejected before Docker is ever invoked.

## Host header

`FRAPPE_SITE_NAME_HEADER` is pinned to `SITE_NAME`. Frappe is multi-tenant and
normally picks the site from the Host header, so a probe hitting a bare IP or an
internal hostname would get "Site does not exist" and fail the check even though
the app is healthy. Pinning it forces every request to resolve to the one site.

## TLS

This stack serves plain HTTP on port 8052. Terminate TLS at the platform's
load balancer. If it forwards `X-Forwarded-For`, set `UPSTREAM_REAL_IP_ADDRESS`
in the compose file to the balancer's address so client IPs are logged correctly.

## Persistence

Four named volumes: `db-data`, `sites`, `logs`, `redis-queue-data`. `db-data`
and `sites` hold all durable state — the database and the site's files,
including uploads and `site_config.json`. Back both up together; they must be
restored as a matched pair. If the platform does not persist named volumes
across deploys, this stack will recreate the site from scratch every time.
