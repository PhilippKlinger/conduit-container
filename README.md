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
- [Django administration](#django-administration)
- [Security notes](#security-notes)
- [Validation](#validation)

## Quickstart

Prerequisites:

- Docker Engine or Docker Desktop with Docker Compose
- Git

The frontend and backend are maintained as separate Git repositories and
included here as submodules. Clone the repository recursively:

```bash
git clone --recurse-submodules git@github.com:PhilippKlinger/conduit-container.git
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

Open `.env`, set unique values for all required blank fields, and configure
the browser-facing addresses for the target Docker host. For a remote host,
the relevant formats are:

```dotenv
API_URL=http://<host-address>:8000/api
DJANGO_ALLOWED_HOSTS=<host-address>
DJANGO_CORS_ORIGIN_WHITELIST=<host-address>:8282
```

`DJANGO_ALLOWED_HOSTS` must not contain a URL scheme or port. The legacy CORS
middleware expects `host:port` without `http://` or `https://`.

Validate the effective configuration and start the complete application:

```bash
docker compose config
docker compose up -d
docker compose ps
```

Open these addresses in a browser:

- frontend: `http://<host-address>:8282`
- backend API: `http://<host-address>:8000/api`
- Django administration: `http://<host-address>:8000/admin/`

## Usage

Use the same commands for routine operation on a local Docker host or VPS:

```bash
docker compose up -d
docker compose ps
docker compose logs --tail 100
docker compose down
```

### Logs

Follow the complete log output or only one service:

```bash
docker compose logs -f
docker compose logs -f backend
```

Save the current logs for later inspection:

```bash
docker compose logs > conduit-container-logs.txt
```

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

Copy `.env.example` to `.env`, set the required values, and do not commit the
resulting file.

| Variable | Purpose | Default value |
| --- | --- | --- |
| `NODE_IMAGE` | Node.js image used to build the Angular application. | `node:20-alpine` |
| `NGINX_IMAGE` | Nginx image used to serve the compiled frontend. | `nginx:1.28.3-alpine` |
| `PYTHON_IMAGE` | Python image compatible with the legacy Django backend. | `python:3.6-slim` |
| `POSTGRES_IMAGE` | PostgreSQL database image. | `postgres:16.14-alpine` |
| `FRONTEND_PORT` | Published frontend host port. | `8282` |
| `BACKEND_PORT` | Published backend host port. | `8000` |
| `API_URL` | Public API base URL used by the browser, including `/api`. | Required; no default |
| `DJANGO_SECRET_KEY` | Unique Django cryptographic signing key. | Required; no default |
| `DJANGO_DEBUG` | Enables Django debug mode. Keep disabled outside local diagnosis. | `False` |
| `DJANGO_ALLOWED_HOSTS` | Comma-separated hosts accepted by Django, without schemes or ports. | Required; no default |
| `DJANGO_CORS_ORIGIN_WHITELIST` | Comma-separated frontend `host:port` values accepted by the legacy CORS middleware. | Required; no default |
| `POSTGRES_DB` | PostgreSQL database name. | `conduit` |
| `POSTGRES_USER` | PostgreSQL application user. | Required; no default |
| `POSTGRES_PASSWORD` | PostgreSQL application password. | Required; no default |
| `DB_WAIT_TIMEOUT` | Maximum time the backend waits for PostgreSQL. | `60` |
| `GUNICORN_WORKERS` | Number of Gunicorn worker processes. | `3` |

`API_URL` is passed to the frontend image as a build argument and compiled
into the Angular bundle. It must use an address reachable by the browser, not
the internal Compose service name `backend`. After changing `API_URL`, rebuild
the frontend:

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

## Django administration

Create an administrator after the services are healthy:

```bash
docker compose exec backend python manage.py createsuperuser
```

The command prompts for the administrator data and passes the supplied
password through Django's normal password hashing. Administrator accounts are
stored in PostgreSQL and remain available while the named database volume is
retained.

Open `http://<host-address>:8000/admin/` and log in with the created account.
Django static files are collected while building the backend image and served
through WhiteNoise.

## Security notes

- Keep `.env`, SSH keys, passwords, tokens, usernames, host addresses,
  database exports, and logs containing sensitive data out of version control.
- Use unique secrets and database credentials for every environment.
- Keep `DJANGO_DEBUG=False` outside local diagnosis.
- PostgreSQL has no published host port and remains inside the Compose
  network.
- The backend runs as the unprivileged `appuser` user.

## Validation

Inspect the service state and application logs:

```bash
docker compose ps
docker compose logs --tail 100 backend frontend database
```

The expected state is:

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
