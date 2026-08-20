# Conduit Container

## Description

This repository runs the Conduit application with Docker Compose:

- Angular frontend served by Nginx
- Django REST backend
- PostgreSQL database with a named persistent volume

The frontend is published on port `8282`. The backend is published on port
`8000` for the browser API and administration interface. PostgreSQL is only
available inside the Compose network.

The frontend and backend are separate Git submodules. Their commits are pinned
by the parent repository so that local builds and CI use a defined source
state.

## Table of Contents

- [Quickstart](#quickstart)
- [Usage](#usage)
- [Configuration](#configuration)
- [CI/CD deployment](#cicd-deployment)
- [Security](#security)
- [Validation](#validation)

## Quickstart

Requirements:

- Git
- Docker Engine or Docker Desktop
- Docker Compose

Clone the repository with its submodules:

```bash
git clone --recurse-submodules git@github.com:PhilippKlinger/conduit-container.git
cd conduit-container
```

For an existing clone, initialize or update the submodules:

```bash
git submodule update --init --recursive
```

Create the local configuration files and replace the required empty values:

```bash
cp .env.example .env
cp .env.backend.example .env.backend
```

Validate the resolved Compose configuration and start the application:

```bash
docker compose config
docker compose up -d
```

Open the application at `http://<host-address>:8282`.

## Usage

Check the service state and restart or stop the application with:

```bash
docker compose ps
docker compose restart
docker compose down
```

View logs for all services or one selected service:

```bash
docker compose logs --tail 100
docker compose logs -f backend
docker compose logs > conduit-container-logs.txt
```

Create a Django administrator after the services are healthy:

```bash
docker compose exec backend python manage.py createsuperuser
```

PostgreSQL data is stored in the named volume `database_data`. A normal
`docker compose down` keeps this volume when containers are recreated.

> [!WARNING]
> `docker compose down -v` removes the database volume and its contents. Use it
> only after an intentional backup or when the data may be deleted.

Changing PostgreSQL credentials after the volume has been initialized requires
a deliberate database credential migration. Editing `.env` alone does not
change existing database accounts.

## Configuration

Copy the example files to `.env` and `.env.backend`. Keep both resulting files
outside version control.

`.env` contains Compose, browser build, and database values:

```dotenv
API_URL=http://localhost:8000/api
FRONTEND_PORT=8282
BACKEND_PORT=8000
POSTGRES_DB=conduit
POSTGRES_USER=<required>
POSTGRES_PASSWORD=<required>
```

`.env.backend` contains Django and backend runtime values:

```dotenv
DJANGO_SECRET_KEY=<required>
DJANGO_DEBUG=False
DJANGO_ALLOWED_HOSTS=localhost,127.0.0.1,[::1]
DJANGO_CORS_ORIGIN_WHITELIST=localhost:8282,127.0.0.1:8282
POSTGRES_USER=<same value as .env>
POSTGRES_PASSWORD=<same value as .env>
```

The main configurable values are:

| Variable | Purpose | Default or requirement |
| --- | --- | --- |
| `NODE_IMAGE` | Node.js image for the Angular build | `node:20-alpine` |
| `NGINX_IMAGE` | Image serving the compiled frontend | `nginx:1.28.3-alpine` |
| `PYTHON_IMAGE` | Python image for the backend | `python:3.6-slim` |
| `POSTGRES_IMAGE` | PostgreSQL image | `postgres:16.14-alpine` |
| `API_URL` | Browser-reachable backend API URL | `http://localhost:8000/api` |
| `FRONTEND_PORT` | Published frontend port | `8282` |
| `BACKEND_PORT` | Published backend port | `8000` |
| `DJANGO_SECRET_KEY` | Django signing key | Required; no default |
| `DJANGO_DEBUG` | Django debug mode | `False` |
| `DJANGO_ALLOWED_HOSTS` | Hosts accepted by Django | Localhost defaults |
| `DJANGO_CORS_ORIGIN_WHITELIST` | Allowed browser origins without URL scheme | Localhost defaults |
| `POSTGRES_DB` | Database name | `conduit` |
| `POSTGRES_USER` | Database user | Required; no default |
| `POSTGRES_PASSWORD` | Database password | Required; no default |
| `POSTGRES_HOST` | Internal database service name | `database` |
| `POSTGRES_PORT` | Internal database port | `5432` |
| `DB_WAIT_TIMEOUT` | Backend database connection timeout | `60` seconds |
| `GUNICORN_WORKERS` | Backend worker count | `3` |

`API_URL` is compiled into the frontend bundle. It must be reachable by the
browser; `backend` is an internal Compose service name and is not valid here.
Rebuild the frontend after changing it:

```bash
docker compose up -d --build frontend
```

Recreate the backend after changing backend runtime values:

```bash
docker compose up -d --force-recreate backend
```

## CI/CD deployment

The GitHub Actions pipeline separates CI from deployment:

1. Pushes to `feature/**`, pushes to `main`, and pull requests targeting `main`
   run CI.
2. CI checks the repository structure, checks out the pinned private
   submodules, builds the frontend, and builds both application images on
   GitHub-hosted runners.
3. On a push to `feature/conduit-deployment`, the images are published to GHCR
   with the source commit as tag and an immutable digest.
4. The reusable deployment workflow validates the image references and connects
   to the staging VPS through SSH.
5. The VPS receives the Compose manifest, pulls the approved images, starts the
   stack with `docker compose up -d --no-build`, and is checked for readiness
   and running image identity.

`main` runs CI but does not trigger this staging deployment. A production
environment would require a separate environment, approval model, and secret
set.

### GitHub configuration

Create the following repository secrets:

| Secret | Scope | Purpose |
| --- | --- | --- |
| `SUBMODULE_TOKEN` | Repository | Fine-grained token with `Contents: Read-only` access to the parent, frontend, and backend repositories |
| `FRONTEND_API_URL` | Repository | Browser-reachable API URL used when the frontend image is built |

Create a GitHub Environment named `staging` and add these environment secrets:

| Secret | Purpose |
| --- | --- |
| `SSH_HOST` | VPS hostname or address |
| `SSH_PORT` | SSH port, normally `22` |
| `SSH_USER` | Deployment user on the VPS |
| `SSH_PRIVATE_KEY` | Private key used only by the GitHub Actions runner |
| `SSH_KNOWN_HOSTS` | Verified host-key entry for the VPS |
| `DEPLOY_PATH` | Absolute Compose project directory on the VPS |

The workflow validates that all required values are present and that host,
port, user, and deployment path have an expected format. Do not put secret
values in workflow files, example files, logs, commits, screenshots, or the
deployment video.

### SSH key and VPS setup

Generate the deployment key outside the repository. Install only its public
key in the deployment user's `~/.ssh/authorized_keys` on the VPS. Store the
private key as the `SSH_PRIVATE_KEY` secret in the `staging` environment.

Obtain the VPS host key through a trusted administration channel, verify its
fingerprint, and store the resulting entry as `SSH_KNOWN_HOSTS`. The workflow
uses strict host-key checking and does not discover or trust a host key during
deployment.

The deployment user must be able to:

- access `DEPLOY_PATH` and write its Compose manifest;
- run Docker and Docker Compose without interactive input;
- reach the Docker daemon;
- use `curl` for readiness checks.

Before the first deployment, create `.env` and `.env.backend` in
`DEPLOY_PATH` on the VPS. They must contain the runtime configuration and be
owned by the deployment user with no group or other permissions. These files
remain on the VPS and are never transferred by the workflow.

### Temporary credentials and deployment files

The workflow writes the private key, known-hosts file, and SSH configuration
only to a restricted directory below `${RUNNER_TEMP}`. Checkout credentials
are not persisted because every checkout uses `persist-credentials: false`.

The Compose manifest is first copied to a uniquely named temporary file below
`DEPLOY_PATH`. It is validated against the approved image digests and moved to
`docker-compose.yaml` only after validation succeeds. Temporary remote files
and runner-side SSH material are removed in cleanup steps that also run after
failures.

The deployment does not build application images on the VPS. The VPS pulls the
approved GHCR images instead. The current packages are publicly pullable, so
the VPS does not log in to GHCR; changing package visibility requires a
separate registry-authentication design.

## Security

- Keep `.env`, `.env.backend`, SSH keys, tokens, passwords, host addresses,
  database exports, and sensitive logs out of Git.
- Use unique credentials and a unique Django secret for each environment.
- Keep `DJANGO_DEBUG=False` outside local diagnosis.
- PostgreSQL has no published host port and is reachable only inside the
  Compose network.
- The backend container runs as the unprivileged `appuser`.
- The deployment uses non-interactive SSH, `IdentitiesOnly`, and strict
  host-key verification.
- Image references are tied to the source commit and published digest.

Image signing, action commit pinning, build caching, production promotion, and
automated rollback are not part of the current deployment path.

## Validation

Validate the local Compose configuration before starting the stack:

```bash
docker compose config
docker compose ps
docker compose logs --tail 100
```

The expected runtime state is:

- the database is healthy;
- the backend is listening on port `8000`;
- Nginx serves the frontend on port `8282`;
- the frontend can load data from the API;
- navigation, registration, login, articles, tags, and administration work
  without API or CORS errors.

Check the API and CORS response from a remote host:

```bash
curl -i \\
  -H "Origin: http://<host-address>:8282" \\
  http://<host-address>:8000/api/tags
```

The response should be successful and include an
`Access-Control-Allow-Origin` value matching the frontend origin.

Check automatic backend restart behavior:

```bash
BACKEND_ID=$(docker compose ps -q backend)
docker compose exec backend sh -c 'kill -TERM 1'
sleep 5
docker inspect --format '{{.RestartCount}}' "${BACKEND_ID}"
docker compose ps
```

The backend should return to `Up` and its restart count should increase.

To verify database persistence, create a uniquely named user or article,
recreate the containers without deleting the volume, and verify that the data
remains available:

```bash
docker compose down
docker compose up -d
```
