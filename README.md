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

This section covers local development. VPS deployments use the production
manifest described in [CI/CD deployment](#cicd-deployment).

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

The commands in this section apply to a local checkout with initialized
submodules. Do not run `docker compose` without `-f docker-compose.prod.yaml`
on the VPS.

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

`API_URL` is compiled into the frontend bundle. Use a browser-reachable URL;
`backend` is an internal Compose service name and is not valid here. Rebuild
the frontend after changing `API_URL`:

```bash
docker compose up -d --build frontend
```

Recreate the backend after changing backend runtime values:

```bash
docker compose up -d --force-recreate backend
```

## CI/CD deployment

The pipeline separates CI from deployment:

1. Pushes to `feature/**`, pushes to `main`, and pull requests targeting `main`
   run CI.
2. CI checks the repository, checks out the pinned submodules, and builds the
   frontend and backend images on GitHub-hosted runners.
3. A push to `feature/conduit-deployment` publishes both images to GHCR with the
   source SHA as tag and an immutable digest.
4. The reusable deployment workflow validates the image references and connects
   to the staging VPS through strict SSH host-key verification.
5. It transfers and validates `docker-compose.prod.yaml`, then runs:

   ```bash
   docker compose -f docker-compose.prod.yaml up -d --no-build
   ```

6. It checks service readiness and the image identity of the running frontend
   and backend containers.

BuildKit caches use separate scopes for frontend and backend. Cache hits reuse
unchanged layers; cache misses perform a normal build. Caches are optional and
do not affect image contents, tags, digests, or deployment correctness.

Local development uses `docker-compose.yaml`, which builds the application
images from the checked-out submodules. Deployment uses
`docker-compose.prod.yaml`, which contains runtime image references only and
does not build application images on the VPS.

On the VPS, the deployment workflow transfers the production manifest and
starts approved GHCR images. It does not require a repository checkout or
`git pull`. Running the default `docker-compose.yaml` there tries to build from
missing submodule sources.

`main` runs CI but does not trigger this staging deployment. Production would
require a separate environment, approval model, and secrets.

### GitHub configuration

Create these repository secrets:

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

The workflow validates required values and formats. Never put secret values in
workflow files, example files, logs, commits, screenshots, or videos.

### SSH key and VPS setup

1. Generate the deployment key outside the repository.
2. Install only the public key in the deployment user's
   `~/.ssh/authorized_keys` on the VPS.
3. Store the private key as `SSH_PRIVATE_KEY` in the `staging` environment.
4. Verify the VPS host-key fingerprint through a trusted channel.
5. Store the verified entry as `SSH_KNOWN_HOSTS`.

The workflow uses strict host-key checking. It never discovers or trusts a new
host key during deployment.

The deployment user needs access to `DEPLOY_PATH`, non-interactive Docker and
Compose access, Docker daemon access, and `curl`.

Before the first deployment, create `.env` and `.env.backend` in `DEPLOY_PATH`.
Keep both files owned by the deployment user with no group or other
permissions. The workflow never transfers these files.

### Temporary credentials and deployment files

The runner stores SSH material only below `${RUNNER_TEMP}` with restricted
permissions. Checkout credentials are not persisted.

Deployment flow:

1. Copy `docker-compose.prod.yaml` to a uniquely named remote temporary file.
2. Resolve its image list with `docker compose config --no-env-resolution`.
3. Compare frontend and backend references with the approved digests.
4. Move the file to `docker-compose.prod.yaml` only after validation succeeds.
5. Remove temporary remote files and runner-side SSH material in cleanup steps.

Compose pulls missing public GHCR images during activation. Private packages
require a separate registry-authentication design.

### Manual VPS recovery

Use this only to recover a stopped deployment. Check the existing containers
and volume first, then provide the approved image references from a successful
workflow run:

```bash
docker ps -a --filter "name=conduit"
docker volume ls --filter "name=conduit"

export FRONTEND_IMAGE='ghcr.io/<owner>/conduit-frontend:<commit>@sha256:<digest>'
export BACKEND_IMAGE='ghcr.io/<owner>/conduit-backend:<commit>@sha256:<digest>'
docker compose -f docker-compose.prod.yaml up -d --no-build
docker compose -f docker-compose.prod.yaml ps
```

Re-running the GitHub Actions deployment is preferred for current images. A
source build on the VPS would require a repository checkout, the correct
branch, initialized submodules, and `docker-compose.yaml`; it is not the
standard deployment path.

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

Image signing, production promotion, and automated rollback are not part of the
current deployment path. GitHub Actions are commit-pinned and image builds use
separate BuildKit caches.

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
