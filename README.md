# Dockerized Django Template

A minimal, Dockerized Django + PostgreSQL + Redis stack, meant to be used
via GitHub's **"Use this template"** button as the starting point for a new
project.

## What this actually is

This repo does **not** ship a pre-built Django project. Only the
infrastructure is hand-written and version-controlled:

- `docker/django/Dockerfile.dev`, `Dockerfile.prod`
- `docker/nginx/`, `docker/caddy/` — reverse proxy for production
- `requirements/base.txt`, `dev.txt`, `prod.txt`
- `.env.dev`, `.env.prod`
- `docker-compose.dev.yml`, `docker-compose.prod.yml`
- `docker/django/settings.overwrite.py` — the Postgres+Redis `settings.py`

The actual Django project (`app/manage.py`, `app/config/`) does not exist in
a fresh checkout — it's generated on demand, via a temporary container, so
you never need Python installed locally and never hand-create any of it.
See **[docs/guide/BOOTSTRAP.md](docs/guide/BOOTSTRAP.md)** for the full,
copy-pasteable command sequence (dev bootstrap through production deploy).

## Stack

- Django, on PostgreSQL + Redis (cache backend)
- `runserver` in dev; `gunicorn` behind **nginx** in production (Caddy
  available as a drop-in alternative — see BOOTSTRAP.md)
- Docker Compose for both environments, healthchecked service startup order

## Using this template

1. Click **Use this template** on GitHub to generate your own repo from
   this one (or `git clone` it directly).
2. Follow **[docs/guide/BOOTSTRAP.md](docs/guide/BOOTSTRAP.md)** start to
   finish — it scaffolds the Django project, wires up Postgres/Redis
   settings, and brings up dev, then walks through a production deploy
   (secrets, reverse proxy choice, healthchecks).

## Quickstart (dev)

```bash
# 1. Scaffold the Django project via a temporary container (no local Python)
docker run --rm -v "$(pwd)/app:/app" -w /app python:3.13-slim \
  bash -c "pip install --quiet django && django-admin startproject config ."
docker run --rm -v "$(pwd)/app:/app" python:3.13-slim \
  chown -R "$(id -u):$(id -g)" /app

# 2. Snapshot the default settings.py, then apply the Postgres+Redis override
cp app/config/settings.py docker/django/settings.default.py
cp docker/django/settings.overwrite.py app/config/settings.py

# 3. Build, start, migrate
docker compose -f docker-compose.dev.yml up --build -d
docker compose -f docker-compose.dev.yml exec api python manage.py migrate
docker compose -f docker-compose.dev.yml exec api python manage.py createsuperuser
```

App: http://localhost:8008/ — Admin: http://localhost:8008/admin

## Documentation

| File | What's in it |
|---|---|
| [docs/guide/BOOTSTRAP.md](docs/guide/BOOTSTRAP.md) | The from-scratch command sequence: dev bootstrap through production deploy (secrets, nginx vs Caddy, healthchecks) |
| [docs/guide/overview.txt](docs/guide/overview.txt) | Why each file/setting is the way it is, plus a running "what was corrected and why" history of bugs found and fixed |
| [docs/guide/database_usage.txt](docs/guide/database_usage.txt) | `psql` access, ORM examples, backup/restore, migrations |
| [docs/guide/redis_usage.txt](docs/guide/redis_usage.txt) | Cache examples, inspecting Redis directly |

## Production

`docker-compose.prod.yml` puts nginx in front of gunicorn (serves
`/static/` directly, is the only service that publishes a host port) and
runs `migrate` + `collectstatic` + `gunicorn` automatically on boot, gated
on Postgres/Redis healthchecks. Before a real deploy: generate real
secrets for `.env.prod`, set `ALLOWED_HOSTS` to your real domain, and
decide on TLS (Caddy gets you automatic HTTPS; nginx needs a certificate
supplied). Full checklist: [docs/guide/BOOTSTRAP.md](docs/guide/BOOTSTRAP.md), section "Step 6 — Production".
