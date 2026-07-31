# Conduit Container

## Description

This repository provides a containerized Conduit application with an Angular
frontend, a Django REST backend, and a PostgreSQL database. Docker Compose
builds both application images, runs all three services in one network, and
stores the database state in a named volume.

The frontend publishes port `8282` by default. The browser communicates
directly with the backend on port `8000`, while PostgreSQL remains available
only inside the Compose network.

## Table of Contents

- [Description](#description)
- [Quickstart](#quickstart)
- [Usage](#usage)
- [Configuration](#configuration)
- [Security notes](#security-notes)
- [Validation](#validation)

## Quickstart

Prerequisites:

- Docker Engine or Docker Desktop with Docker Compose
- Git

The frontend and backend are maintained as separate Git repositories and
included here as submodules. Clone the repository recursively:

```bash
git clone \
  --branch feature/server-setup \
  --recurse-submodules \
  git@github.com:PhilippKlinger/conduit-container.git
cd conduit-container
```

For an existing clone without initialized submodules, run:

```bash
git submodule update --init --recursive
```

Create the local environment file:

```bash
cp .env.example .env
```

Open `.env`, set unique values for all required blank fields, and review the
host-specific values described in [Configuration](#configuration).

Validate the effective configuration and start the complete application:

```bash
docker compose config
docker compose up -d
```

Open these addresses in a browser:

- frontend: `http://<host-address>:8282`
- backend API: `http://<host-address>:8000/api`
- Django administration: `http://<host-address>:8000/admin/`

## Usage

Use the same commands for routine operation on a local Docker host or VPS:

```bash
docker compose ps
docker compose restart
docker compose down
```

Create a Django administrator after the services are healthy:

```bash
docker compose exec backend python manage.py createsuperuser
```

Open `http://<host-address>:8000/admin/` and log in with the created account.

### Logs

Inspect recent logs, follow one service, or save the current output:

```bash
docker compose logs --tail 100
docker compose logs -f backend
docker compose logs > conduit-container-logs.txt
```

Without a service name, Compose displays logs from all services. Add
`frontend`, `backend`, or `database` to limit the output.

### Data persistence

PostgreSQL stores users, articles, comments, tags, and other application data
in the `database_data` named volume mounted at `/var/lib/postgresql/data`.
Docker retains this volume after a normal `docker compose down`, container
recreation, or image rebuild.

> [!WARNING]
> Do not use `docker compose down -v` or remove the `database_data` volume
> unless the database has been intentionally backed up and its contents may be
> deleted.

Named volumes belong to one Docker host. Moving the application to another
machine requires a separate database backup and restore.

## Configuration

Set the required values in the untracked `.env` file. `PUBLIC_HOST` is the
single place where the externally reachable address is configured:

```dotenv
PUBLIC_HOST=<host-address>
```

Use `localhost` for a local run. Enter the host or IP only, without a URL
scheme and without a port. Compose derives three values from it: the API URL
compiled into the frontend bundle, Django's `ALLOWED_HOSTS`, and the whitelist
for the legacy CORS middleware, which expects `host:port` without `http://`
or `https://`.

| Variable | Purpose | Default value |
| --- | --- | --- |
| `NODE_IMAGE` | Node.js image used to build the Angular application. | `node:20-alpine` |
| `NGINX_IMAGE` | Nginx image used to serve the compiled frontend. | `nginx:1.28.3-alpine` |
| `PYTHON_IMAGE` | Python image compatible with the legacy Django backend. | `python:3.6-slim` |
| `POSTGRES_IMAGE` | PostgreSQL database image. | `postgres:16.14-alpine` |
| `PUBLIC_HOST` | Host or IP the browser uses to reach this deployment. The API URL, `ALLOWED_HOSTS`, and the CORS whitelist derive from it. | `localhost` |
| `FRONTEND_PORT` | Published frontend host port. | `8282` |
| `BACKEND_PORT` | Published backend host port. | `8000` |
| `DJANGO_SECRET_KEY` | Unique Django cryptographic signing key. | Required; no default |
| `DJANGO_DEBUG` | Enables Django debug mode. Keep disabled outside local diagnosis. | `False` |
| `POSTGRES_DB` | PostgreSQL database name. | `conduit` |
| `POSTGRES_USER` | PostgreSQL application user. | Required; no default |
| `POSTGRES_PASSWORD` | PostgreSQL application password. | Required; no default |
| `DB_WAIT_TIMEOUT` | Maximum time in seconds the backend waits for PostgreSQL. | `60` |
| `GUNICORN_WORKERS` | Number of Gunicorn worker processes. | `3` |

`PUBLIC_HOST` becomes part of the API URL that Compose passes to the frontend
image as a build argument, and Angular compiles that URL into the bundle. It
must therefore be an address the browser can reach, not the internal Compose
service name `backend`. After changing it, rebuild the frontend:

```bash
docker compose up -d --build frontend
```

The backend reads its Django and database values when its container starts.
Recreate the backend after changing those values:

```bash
docker compose up -d --force-recreate backend
```

Changing PostgreSQL credentials after the named volume has already been
initialized requires a deliberate database credential migration. Editing
`.env` alone does not update existing database accounts.

## Security notes

- Keep `.env`, SSH keys, passwords, tokens, usernames, host addresses,
  database exports, and logs containing sensitive data out of version control.
- Use unique secrets and database credentials for every environment.
- Keep `DJANGO_DEBUG=False` outside local diagnosis.
- PostgreSQL has no published host port and remains inside the Compose
  network.
- The backend runs as the unprivileged `appuser` user.

## Validation

Use the status and log commands from [Usage](#usage). The expected state is:

- the database is `healthy`;
- the backend logs show successful migrations and Gunicorn listening on port
  `8000`;
- the frontend logs show Nginx ready to serve the Angular application;
- the frontend is reachable on host port `8282`;
- navigation, registration, login, article operations, tags, and Django
  administration work without API or CORS errors.

Verify CORS from a remote host:

```bash
curl -i \
  -H "Origin: http://<host-address>:8282" \
  http://<host-address>:8000/api/tags
```

The response must return HTTP `200` and an `Access-Control-Allow-Origin`
header matching the frontend origin.

Verify automatic backend restart behavior:

```bash
BACKEND_ID=$(docker compose ps -q backend)
docker compose exec backend sh -c 'kill -TERM 1'
sleep 5
docker inspect --format '{{.RestartCount}}' "${BACKEND_ID}"
docker compose ps
```

The backend must return to `Up`, and its restart count must increase.

To verify persistence, create a uniquely named article and user through the
application, recreate the containers without deleting the volume, and confirm
that both records remain:

```bash
docker compose down
docker compose up -d
```
