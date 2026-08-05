# ERPNext All-in-One

A single custom Docker image bundling **ERPNext** with a curated set of
official Frappe apps and a Malaysia LHDN e-Invoicing integration — built
straight from source via GitHub Actions, so nothing disappears when a
container gets recreated.

[![Build and Push](https://github.com/Amirul78800/erpnext-allinone/actions/workflows/build.yml/badge.svg)](https://github.com/Amirul78800/erpnext-allinone/actions/workflows/build.yml)
[![Docker Hub](https://img.shields.io/badge/docker%20hub-allinone--erpnext-2496ED?logo=docker&logoColor=white)](https://hub.docker.com/r/amirul123/allinone-erpnext)

## Why this exists

Frappe apps installed at runtime with `bench get-app` live only inside a
container's writable layer — they vanish the moment that container is
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
| **CRM** | `main` | Standalone sales pipeline, modern UI |
| **Helpdesk** | `main` | Support ticketing |
| **Insights** | `develop` | BI dashboards, cross-app reporting |
| **Builder** | `develop` | No-code website/portal builder |
| **MyInvois** ([ERPGulf](https://github.com/ERPGulf/myinvois)) | `main` | Malaysia LHDN e-Invoice compliance |

> **Note on installing MyInvois:** the repo is named `myinvois`, but the
> actual Frappe app/module inside it is `myinvois_erpgulf`. Use that name
> with `bench install-app`, not `myinvois`.
>
> **Note on Telephony:** you won't need to pass `--install-app telephony`
> explicitly — Frappe auto-installs it as a required dependency the moment
> CRM or Helpdesk installs, as long as the code is present in the image
> (which it is here).

**Base:** Python 3.11, Node 20, built on `frappe/frappe_docker`'s layered
Containerfile.

## Branching

This repo doesn't assume `main` is the only buildable branch. The workflow
triggers on push to **any** branch and tags the resulting image with that
branch name (e.g. a push to `2.0` produces
`amirul123/allinone-erpnext:2.0`). The `:latest` tag is only updated on
pushes to `main`, so you can iterate on a feature/testing branch without
touching what's already deployed. Once a branch is stable, merge it into
`main` to promote it to `:latest`.

## Repo structure

```
erpnext-allinone/
├── .github/
│   └── workflows/
│       └── build.yml
├── apps.json
├── DOCKERHUB_OVERVIEW.md
└── README.md
```

## One-time setup

1. In this repo's Settings → Secrets and variables → Actions, add:
   - `DOCKERHUB_USERNAME` — your Docker Hub username
   - `DOCKERHUB_TOKEN` — a Docker Hub access token (Account Settings →
     Security → New Access Token — not your password)
2. Push to any branch, or trigger manually from the Actions tab.

## What the workflow does

- Checks out this repo (for `apps.json`)
- Checks out `frappe/frappe_docker` upstream (owns the build file, which is
  named `Containerfile`, not `Dockerfile`)
- Builds one image with every app above baked in, with `apps.json` passed as
  a **BuildKit secret** (not a build-arg), so it never leaks into the
  image's layer history
- Pushes to Docker Hub, tagged by branch name, commit SHA, and (on `main`)
  `latest`

Expect ~20-30 minutes per build on GitHub's free runners — this many apps
means a lot of `bench build` asset compilation.

## Using the image

```yaml
services:
  backend:
    image: amirul123/allinone-erpnext:latest
    # ...same pattern for websocket, queue-long, queue-short, scheduler
```

First-time site creation:

```bash
bench new-site yoursite.local \
  --mariadb-root-password <password> \
  --admin-password <password> \
  --mariadb-user-host-login-scope=% \
  --install-app erpnext \
  --install-app hrms \
  --install-app crm \
  --install-app helpdesk \
  --install-app insights \
  --install-app builder \
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
  --build-arg=PYTHON_VERSION=3.11 \
  --build-arg=NODE_VERSION=20 \
  --secret=id=apps_json,src=/path/to/apps.json \
  --tag=allinone-erpnext:test \
  --file=images/layered/Containerfile .
```

Requires Docker Engine v23.0+ (BuildKit default).

## License

Each bundled app retains its own upstream license (mostly MIT/GPL-3.0). This
repo is just build tooling — see each app's own repository for licensing
details. Not affiliated with or endorsed by Frappe Technologies Pvt. Ltd. or
ERPGulf.
