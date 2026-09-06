# Obico Server for CapRover

This repository contains the configuration to deploy the [Obico server](https://github.com/TheSpaghettiDetective/obico-server) on [CapRover](https://caprover.com/) using Docker Compose and GitHub Actions for automated image builds.

## Overview

The Obico server is split into three services:
- **Backend**: Django application handling the main Obico functionality
- **Worker**: Celery worker for background tasks
- **ML API**: Separate service for machine learning operations (spaghetti detection)

This setup expects external PostgreSQL and Redis services, which should be deployed as separate one-click apps on CapRover.

## Prerequisites

Before deploying this stack, you need to deploy the following services on your CapRover instance:

1. **PostgreSQL** (using the one-click app)
   - Set environment variables:
     - `POSTGRES_PASSWORD`: A secure password
     - `POSTGRES_USER`: `obico` (or any user)
     - `POSTGRES_DB`: `obico` (or any database name)
   - Note the internal service name (e.g., `postgres`) and port (usually 5432)

2. **Redis** (using the one-click app)
   - No special configuration needed beyond the default
   - Note the internal service name (e.g., `redis`) and port (usually 6379)

## Deployment Steps

1. **Deploy PostgreSQL and Redis** as one-click apps on CapRover.
2. **Configure CapRover to access GHCR**:
   - Go to your CapRover dashboard → Registry → Add Registry
   - Registry URL: `ghcr.io`
   - Username: Your GitHub username (or the account that owns the images)
   - Password: A GitHub Personal Access Token with `read:packages` scope
3. **Deploy the Obico stack**:
   - In CapRover, go to Apps → Deploy Docker Compose
   - Paste the contents of [`docker-compose.yml`](docker-compose.yml) into the form
   - Click Deploy
4. **Configure environment variables** for the Obico app in CapRover:
   - After deployment, go to the app's settings → Environment Variables
   - Set the following variables (adjust values based on your PostgreSQL and Redis deployments):
     - `DATABASE_URL`: `postgres://obico:YOUR_POSTGRES_PASSWORD@postgres:5432/obico`
     - `REDIS_URL`: `redis://redis:6379`
     - Optional but recommended:
       - `DEFAULT_FROM_EMAIL`: `noreply@yourdomain.com`
       - `EMAIL_HOST`, `EMAIL_HOST_USER`, `EMAIL_HOST_PASSWORD`, `EMAIL_PORT`, `EMAIL_USE_TLS` (if using email)
       - `SITE_USES_HTTPS`: `True` (if you have SSL set up in CapRover)
       - `SITE_IS_PUBLIC`: `True` (if you want to allow signups)
       - `ACCOUNT_ALLOW_SIGN_UP`: `True` (if you want to allow new users to sign up)

## How it works

- The `docker-compose.yml` file references images hosted on GitHub Container Registry (GHCR):
  - `ghcr.io/<your-username>/obico-backend:latest`
  - `ghcr.io/<your-username>/obico-ml_api:latest`
- The GitHub Actions workflow (`.github/workflows/obico-ci.yml`) automatically builds and pushes these images whenever code is pushed to the `main` branch.
- The Obico source is included as a Git submodule at `source/`.

## Updating the stack

To update to the latest version:
1. Push changes to the `main` branch of this repository (or just make a trivial commit to trigger the workflow).
2. Wait for the GitHub Actions workflow to complete (check the Actions tab).
3. In CapRover, go to your Obico app → Update/Rebuild → Check for updates (or simply restart the app to pull the latest images).

## Notes

- The frontend service is not a separate container; the frontend files are mounted into the backend container at `/frontend` for serving by Daphne.
- The ML API service is built from the `ml_api` directory in the Obico source.
- All services use the `unless-stopped` restart policy.

## Troubleshooting

- If the app fails to start, check the logs in CapRover for each service.
- Ensure that the `DATABASE_URL` and `REDIS_URL` are correctly pointing to your PostgreSQL and Redis services.
- Make sure the CapRover instance can pull images from GHCR (registry configuration step above).

## License

This configuration is provided as-is. The Obico server is licensed under the AGPL-3.0 license. See the source repository for details.
