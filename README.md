# Obico Server for CapRover

This repository provides a CapRover One-Click deployment for the [Obico server](https://github.com/TheSpaghettiDetective/obico-server).

The goal is to make a complete self-hosted Obico installation deployable from a single CapRover template without manually wiring service names between separate apps.

## What gets deployed

The template creates five CapRover services from one base app name:

- **PostgreSQL** – persistent Obico database
- **Redis** – persistent cache / queue backend
- **Backend** – Django + Daphne web application
- **Worker** – Celery worker and beat scheduler
- **ML API** – Obico failure-detection service

For example, if the base app name is `obico`, CapRover creates services following this pattern:

```text
obico-postgres
obico-redis
obico-backend
obico-worker
obico-ml-api
```

The services reference each other through CapRover's internal service names, for example:

```text
srv-captain--obico-postgres
srv-captain--obico-redis
srv-captain--obico-ml-api
```

This means the deployment remains consistent even when the chosen base app name changes.

## Images

The Obico application images are built automatically by GitHub Actions and published publicly to GitHub Container Registry:

```text
ghcr.io/tylonhh/obico-backend:latest
ghcr.io/tylonhh/obico-ml-api:latest
```

PostgreSQL and Redis use official upstream images and are not rebuilt by this repository.

Current bundled versions:

```text
postgres:16-alpine
redis:7.2-alpine
```

Because the GHCR images are public, no GHCR credentials are required in CapRover.

The One-Click template references these images directly. CapRover does not build derivative backend, worker, ML API, or Redis images during installation. This avoids long in-dashboard image builds and reduces the chance of proxy/API timeouts during One-Click deployment.

## Deployment

### 1. Open the CapRover One-Click template screen

In CapRover go to:

```text
Apps
→ One-Click Apps / Databases
→ >> TEMPLATE <<
```

### 2. Paste the template

Copy the complete contents of [`docker-compose.yml`](docker-compose.yml) into the template field and deploy it.

The file uses CapRover One-Click v4 syntax:

```yaml
captainVersion: 4
```

### 3. Choose the base app name

For example:

```text
obico
```

CapRover then derives all five service names automatically.

### 4. Configure the generated values

The template generates secure random values for:

- PostgreSQL password
- Redis password
- Django secret key
- ML API authentication token

Optional SMTP settings can also be entered during installation or changed later in CapRover.

### 5. Configure the public backend

After deployment, open the app ending in:

```text
-backend
```

For example:

```text
obico-backend
```

Then:

1. Attach your desired domain.
2. Enable HTTPS.
3. Verify that the container HTTP port is `3334`.

Only the backend should normally be exposed publicly.

PostgreSQL, Redis, worker and ML API are configured as internal-only services.

## Persistence

The template creates persistent CapRover volumes for PostgreSQL and Redis:

```text
<appname>-postgres-data
<appname>-redis-data
```

Recreating application containers therefore does not delete database or Redis data as long as the volumes themselves are retained.

Always back up PostgreSQL before destructive upgrades or removing the deployment.

## Internal connections

The template automatically builds the connection strings from the chosen app name.

Database connection pattern:

```text
postgres://obico:<generated-password>@srv-captain--<appname>-postgres:5432/obico
```

Redis connection pattern:

```text
redis://:<generated-password>@srv-captain--<appname>-redis:6379
```

The backend connects to the ML API through:

```text
http://srv-captain--<appname>-ml-api:3333
```

No manual service-name editing should be necessary.

## Service startup commands

Current CapRover versions support the `command` field in the One-Click / Compose parser, so the template uses the published images directly and supplies each service's runtime command without rebuilding an image in CapRover.

### Backend

```text
python manage.py migrate
python manage.py collectstatic -v 2 --noinput
daphne -b 0.0.0.0 -p 3334 config.routing:application
```

### Worker

```text
celery -A config worker --beat -l info -c 2 -Q realtime,celery
```

### ML API

```text
gunicorn --bind 0.0.0.0:3333 --workers 1 wsgi
```

### Redis

Redis is started with AOF persistence and password authentication enabled:

```text
redis-server --appendonly yes --requirepass <generated-password>
```

These commands match the corresponding upstream Obico services while avoiding unnecessary CapRover-side Docker builds.

## GitHub Actions

### Image build workflow

The image workflow is located at:

```text
.github/workflows/obico-ci.yml
```

It runs on:

- pushes to `master`
- pull requests targeting `master`
- manual `workflow_dispatch`

The workflow builds:

```text
ghcr.io/tylonhh/obico-backend
ghcr.io/tylonhh/obico-ml-api
```

Only a direct push to `master` publishes images to GHCR. Pull requests and manual validation runs build the images without publishing them.

Published tags are:

```text
ghcr.io/tylonhh/obico-backend:<commit-sha>
ghcr.io/tylonhh/obico-backend:latest

ghcr.io/tylonhh/obico-ml-api:<commit-sha>
ghcr.io/tylonhh/obico-ml-api:latest
```

GitHub Actions cache is enabled separately for the backend and ML API Buildx builds. This significantly reduces rebuild time when Docker layers have not changed.

Concurrency control also cancels obsolete builds for the same workflow/ref when a newer build starts.

### Automatic upstream update workflow

The upstream update workflow is located at:

```text
.github/workflows/update-obico.yml
```

The `source/` Git submodule explicitly tracks Obico's upstream `release` branch.

Once per day, GitHub Actions checks whether the pinned Obico commit is behind the latest upstream `release` commit.

If no update exists, nothing is changed.

If an update exists, the workflow:

1. updates the `source` submodule on branch `chore/update-obico`
2. creates or refreshes a pull request targeting `master`
3. manually dispatches the Docker build workflow against that update branch
4. validates both images without publishing them

After the update pull request is reviewed and merged, the normal `master` push triggers the production build and publishes the new `latest` and commit-SHA images.

This keeps upstream updates automated while avoiding silent production upgrades.

## Updating Obico manually

The Obico source is included as a Git submodule under:

```text
source/
```

It is configured in `.gitmodules` to track:

```text
branch = release
```

Normally the scheduled update workflow handles upstream detection automatically.

To update manually:

1. Update `source` to the desired upstream Obico revision.
2. Commit the changed submodule reference.
3. Open/merge the change into `master`.
4. Wait for the GitHub Actions build to finish successfully.
5. Redeploy or force-update the backend, worker and ML API apps in CapRover so they pull the newest `latest` images.

For reproducible production deployments, consider replacing `latest` in the template with a fixed image tag or commit SHA.

## Environment variables

The template configures the core variables required for communication between the services, including:

```text
DATABASE_URL
REDIS_URL
INTERNAL_MEDIA_HOST
ML_API_HOST
ML_API_TOKEN
DJANGO_SECRET_KEY
SITE_USES_HTTPS
SITE_IS_PUBLIC
ACCOUNT_ALLOW_SIGN_UP
WEBPACK_LOADER_ENABLED
```

SMTP-related settings are optional and can be changed after deployment from the backend app's CapRover environment-variable settings.

Additional Obico variables can be added later if needed, such as integrations, notifications, LLM/VLM providers, Telegram, Twilio, Sentry or other optional upstream features.

## Troubleshooting

### One-Click deployment returns HTTP 504

A 504 from the CapRover dashboard means the dashboard/API request timed out. Check the CapRover Apps list before retrying because some services may still have been created successfully.

The current template avoids CapRover-side `dockerfileLines` builds and pulls prebuilt images directly, which makes this substantially less likely than previous versions of the template.

If a 504 still occurs, check server resources, CapRover logs, Docker pull progress and whether the host can pull `ghcr.io/tylonhh/obico-ml-api:latest` directly.

### Backend does not start

Check the backend logs first. Typical causes include:

- PostgreSQL not ready yet
- invalid environment variables
- database migration failure
- image pull failure

### Worker cannot connect

Confirm that the backend and worker use the same generated database, Redis and ML API values.

### ML API does not respond

The ML API listens internally on port `3333` and should normally not be exposed publicly.

The backend uses the generated shared `ML_API_TOKEN` when communicating with it.

### Database connection errors

The PostgreSQL hostname should follow this internal pattern:

```text
srv-captain--<appname>-postgres
```

Do not use the public CapRover domain for database traffic.

### Redis connection errors

The Redis hostname should follow this internal pattern:

```text
srv-captain--<appname>-redis
```

Redis authentication is enabled by this template.

## Repository layout

```text
.github/workflows/obico-ci.yml       Docker image build and publish workflow
.github/workflows/update-obico.yml   Scheduled upstream Obico update workflow
.gitmodules                          Obico upstream release-branch tracking
docker-compose.yml                   CapRover One-Click v4 template
source/                              Obico upstream source submodule
README.md                            Deployment documentation
```

## Upstream projects

- Obico Server: https://github.com/TheSpaghettiDetective/obico-server
- CapRover: https://caprover.com/
- CapRover One-Click Apps: https://github.com/caprover/one-click-apps

## License

This repository contains deployment configuration for Obico.

Obico itself is licensed under the AGPL-3.0 license. Refer to the upstream Obico repository for its source code and license terms.
