# ERPNext All-in-One

A single custom Docker image bundling **ERPNext** with a curated set of
official Frappe apps and a Malaysia LHDN e-Invoicing integration - built
straight from source via GitHub Actions, so nothing disappears when a
container gets recreated.

[![Build and Push](https://github.com/Amirul78800/erpnext-allinone/actions/workflows/build.yml/badge.svg)](https://github.com/Amirul78800/erpnext-allinone/actions/workflows/build.yml)
[![Docker Hub](https://img.shields.io/badge/docker%20hub-allinone--erpnext-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/amirul123/allinone-erpnext)

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

## Quick Start with Docker Compose

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

## Local test build

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

## License

Each bundled app retains its own upstream license (mostly MIT/GPL-3.0). This
repo is just build tooling - see each app's own repository for licensing
details. Not affiliated with or endorsed by Frappe Technologies Pvt. Ltd. or
ERPGulf.