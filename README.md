# ERPNext All-in-One

A single custom Docker image bundling **ERPNext** with a curated set of
official Frappe apps and a Malaysia LHDN e-Invoicing integration - built
straight from source via GitHub Actions, so nothing disappears when a
container gets recreated.

[![Build and Push All-in-One ERPNext Image](https://github.com/Amirul78800/erpnext-allinone/actions/workflows/build.yml/badge.svg)](https://github.com/Amirul78800/erpnext-allinone/actions)
[![Docker Hub](https://img.shields.io/badge/dockerhub-allinone--erpnext-blue?logo=docker&style=flat-square)](https://hub.docker.com/r/amirul123/allinone-erpnext)
[![Docker Pulls](https://img.shields.io/docker/pulls/amirul123/allinone-erpnext?style=flat-square)](https://hub.docker.com/r/amirul123/allinone-erpnext)
[![Docker Image Size](https://img.shields.io/docker/image-size/amirul123/allinone-erpnext/latest?style=flat-square)](https://hub.docker.com/r/amirul123/allinone-erpnext)
[![License](https://img.shields.io/github/license/Amirul78800/erpnext-allinone?style=flat-square)](LICENSE)
[![GitHub release](https://img.shields.io/github/v/release/Amirul78800/erpnext-allinone?style=flat-square)](https://github.com/Amirul78800/erpnext-allinone/releases)

## Why this exists

Frappe apps installed at runtime with `bench get-app` live only inside a
container's writable layer - they vanish the moment that container is
recreated (`docker compose down && up`, an image update, a host migration).
This repo bakes every app directly into the image at build time, so the
whole stack survives container recreation. Only the site's own data (in a
separate volume) needs to persist, which you'd be doing anyway.

## Apps included

| App | Branch | Purpose |
|---|---|---|
| **Frappe** (core) | `version-16` | Base framework |
| **ERPNext** | `version-16` | Accounting, inventory, sales, purchasing |
| **HRMS** | `version-16` | Attendance, leave, payroll |
| **Telephony** | `develop` | Required dependency of CRM & Helpdesk (call/SMS logging) |
| **WhatsApp** | `develop` | Official WhatsApp integration for messaging and notifications |
| **CRM** | `main` | Standalone sales pipeline, modern UI |
| **Helpdesk** | `main` | Support ticketing |
| **Insights** | `version-3` | BI dashboards, cross-app reporting |
| **Builder** | `develop` | No-code website/portal builder |
| **MyInvois** ([ERPGulf](https://github.com/ERPGulf/myinvois)) | `main` | Malaysia LHDN e-Invoice compliance |

> **Note on installing MyInvois:** the repo is named `myinvois`, but the
> actual Frappe app/module inside it is `myinvois_erpgulf`. Use that name
> with `bench install-app`, not `myinvois`.
>
> **Note on Telephony:** you won't need to pass `--install-app telephony`
> explicitly - Frappe auto-installs it as a required dependency the moment
> CRM or Helpdesk installs, as long as the code is present in the image
> (which it is here).

**Base:** Python 3.14, Node 24, built on `frappe/frappe_docker`'s layered
Containerfile.

## Branching & Tags

The workflow triggers on push to **any** branch and pushes the following image tags to Docker Hub:
- `<image>:<branch_name>` — e.g. a push to `2.0` produces `amirul123/allinone-erpnext:2.0`
- `<image>:<sha>` — exact commit SHA for immutable deployment pins

When pushing to **`main`**, the workflow additionally publishes:
- `<image>:v16` and `<image>:latest` — stable aliases
- `<image>:v16-YYYYMMDD` — dated snapshot tag

## Repo structure

```
erpnext-allinone/
├── .github/
│   └── workflows/
│       └── build.yml
├── apps.json
├── compose.yaml
├── example.env
└── README.md
```

## One-time setup

1. In this repo's **Settings → Secrets and variables → Actions**, add:
   - `DOCKERHUB_USERNAME` - your Docker Hub username
   - `DOCKERHUB_TOKEN` - a Docker Hub access token (Account Settings → Security → New Access Token - not your password)
2. Push to any branch, or trigger manually from the Actions tab.

# Quick Start with Docker Compose

1. Copy the sample environment file:
   ```bash
   cp example.env .env
   ```
2. Edit `.env` to set your site name and database passwords.
3. Start the containers:
   ```bash
   docker compose up -d
   ```
4. Create your site and install all baked-in apps:
   ```bash
   docker compose exec backend bench new-site yoursite.local \
     --mariadb-root-password <password> \
     --admin-password <password> \
     --mariadb-user-host-login-scope=% \
     --install-app erpnext \
     --install-app hrms \
     --install-app crm \
     --install-app helpdesk \
     --install-app insights \
     --install-app builder \
     --install-app whatsapp \
     --install-app myinvois_erpgulf
   ```

`--mariadb-user-host-login-scope=%` avoids a subtle failure mode: without
it, the database user is locked to the container's IP at creation time, and
the first container restart (new IP) breaks every DB connection with
`Access denied`.

5.(Optional) Install the required modules for your deployment only (if you already run install-app erpnext):
```bash
# HR & Payroll
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app hrms

# Sales CRM & Customer Service (telephony dependency is handled automatically)
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app crm
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app helpdesk

# Communications & Reporting
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app whatsapp
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app insights
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app builder

# Malaysia LHDN MyInvois Compliance
docker compose exec backend bench --site ${SITE_NAME:-frontend} install-app myinvois_erpgulf
```

# Environment Variables (`.env`)

Copy the template from `example.env` to `.env` before starting the Docker Compose cluster:

```bash
cp example.env .env
```
| Variable | Example Value | Description |
| :--- | :--- | :--- |
| SITE_NAME | erp.yourdomain.com / frontend | Target Frappe site name |
| DB_PASSWORD | db_secret_pass | Password for the MariaDB database user |
| DB_ROOT_PASSWORD | mariadb_root_pass | MariaDB root administrative password |
| ADMIN_PASSWORD | admin_super_secret | Initial password for the Frappe Administrator account |
| CONTAINER_PORT | 8080 | Host port mapped to the web server / reverse proxy |
| LETSENCRYPT_EMAIL | admin@yourdomain.com | Email for automated Let's Encrypt SSL certificates (if using Traefik) |

# Operations & Maintenance

## Database & File Backups 
Run backups manually or via cron. This exports the database dump and archives public/private uploaded files into sites/${SITE_NAME}/private/backups/:
```bash
docker compose exec backend bench --site ${SITE_NAME:-frontend} backup --with-files
```
## Site Restoration
To restore a site from an existing .sql.gz archive:
```bash
docker compose exec backend bench --site ${SITE_NAME:-frontend} restore \
  /path/to/backup.sql.gz \
  --mariadb-root-password ${DB_ROOT_PASSWORD}
```
## Upgrade & Migration Workflow
When updating to a newer image release, pull the image and apply migrations without data loss:
```bash
# 1. Pull latest images
docker compose pull

# 2. Recreate containers with new image
docker compose up -d

# 3. Apply database schema migrations
docker compose exec backend bench --site ${SITE_NAME:-frontend} migrate

# 4. Clear cache to reflect frontend asset updates
docker compose exec backend bench --site ${SITE_NAME:-frontend} clear-cache
```
## Useful Bench Commands
Access Frappe CLI utilities directly within the backend container:

### Interactive Bench Shell:
```bash
docker compose exec backend bash
```
### Frappe Python Console:
```bash
docker compose exec backend bench --site ${SITE_NAME:-frontend} console
```
### List Installed Applications:
```bash
docker compose exec backend bench --site ${SITE_NAME:-frontend} list-apps
```
### Restart Background Workers:
```bash
docker compose restart queue-short queue-long queue-default
```
# Local test build

```bash
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --build-arg=PYTHON_VERSION=3.14.0 \
  --build-arg=NODE_VERSION=24.12.0 \
  --secret=id=apps_json,src=/path/to/apps.json \
  --tag=allinone-erpnext:test \
  --file=images/layered/Containerfile .
```

Requires Docker Engine v23.0+ (BuildKit default).
# License

Each bundled app retains its own upstream license (mostly MIT/GPL-3.0). This
repo is just build tooling - see each app's own repository for licensing
details. Not affiliated with or endorsed by Frappe Technologies Pvt. Ltd. or
ERPGulf.
