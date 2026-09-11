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

1. Enable the app as a web app if required.
2. Attach your desired domain.
3. Enable HTTPS.
4. Verify that the container HTTP port is `3334`.

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

CapRover's Docker Compose parser does not reliably support every standard Compose field, including arbitrary `command` handling in imported templates.

For that reason the One-Click template uses `caproverExtra.dockerfileLines` to create small derivative images with the required startup commands.

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

These match the corresponding services in the upstream Obico Docker Compose configuration.

## GitHub Actions

The workflow is located at:

```text
.github/workflows/obico-ci.yml
```

It runs on pushes to the repository's default branch:

```text
master
```

and builds/publishes:

```text
ghcr.io/tylonhh/obico-backend:<commit-sha>
ghcr.io/tylonhh/obico-backend:latest

ghcr.io/tylonhh/obico-ml-api:<commit-sha>
ghcr.io/tylonhh/obico-ml-api:latest
```

Pull requests build the images for validation but do not push them.

## Updating Obico

The Obico source is included as a Git submodule under:

```text
source/
```

To update Obico:

1. Update the `source` submodule to the desired upstream Obico revision.
2. Commit and push the changed submodule reference to `master`.
3. Wait for the GitHub Actions build to finish successfully.
4. Redeploy or force-update the backend, worker and ML API apps in CapRover so they pull the newest `latest` images.

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

### Backend does not start

Check the backend logs first. Typical causes include:

- PostgreSQL not ready yet
- invalid environment variables
- database migration failure
- image pull/build failure

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
.github/workflows/obico-ci.yml   GitHub Actions image build
.gitmodules                      Obico upstream submodule configuration
docker-compose.yml               CapRover One-Click v4 template
source/                          Obico upstream source submodule
README.md                        Deployment documentation
```

## Upstream projects

- Obico Server: https://github.com/TheSpaghettiDetective/obico-server
- CapRover: https://caprover.com/
- CapRover One-Click Apps: https://github.com/caprover/one-click-apps

## License

This repository contains deployment configuration for Obico.

Obico itself is licensed under the AGPL-3.0 license. Refer to the upstream Obico repository for its source code and license terms.
