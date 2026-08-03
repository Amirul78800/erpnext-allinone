# Fortuntech ERPNext Custom Image

Single custom Docker image built on `frappe_docker`, with all apps baked in at
build time so they survive container recreation — no more manual `bench get-app`
after every restart.

## App list (Production deployment)

| App | Branch | Covers |
|---|---|---|
| frappe (core) | version-16 | framework, auto-included |
| erpnext | version-16 | Inventory/warehouse, stock ledger |
| hrms | version-16 | Clock in/out, leave application + approval |
| crm | main | Sales pipeline (standalone, modern UI) |
| helpdesk | main | Support ticketing |
| insights | main | BI/dashboards |
| builder | main | No-code portal/landing pages |
| myinvois | main | LHDN e-Invoice (Malaysia compliance) |

Confirmed working: MyInvois on Frappe v16 (tested by Amirul directly), so
everything is pinned to `version-16` / `main` — one image, no variants.

## Repo structure (flat, root level)

```
erpnext-allinone/
├── .github/
│   └── workflows/
│       └── build.yml
├── apps.json
└── README.md
```

## One-time setup

1. Push this repo with the flat structure above to GitHub
   (e.g. `Amirul78800/erpnext-allinone`).
2. Repo Settings → Secrets and variables → Actions, add:
   - `DOCKERHUB_USERNAME` = `amirul123`
   - `DOCKERHUB_TOKEN` = Docker Hub access token (Account Settings → Security →
     New Access Token, not your password)
3. Push to `main`, or Actions tab → "Build and Push Fortuntech ERPNext Image" →
   Run workflow.

## What the workflow does

- Checks out this repo (for `apps.json`)
- Checks out `frappe/frappe_docker` upstream (owns the build file, named
  `Containerfile` — not `Dockerfile`)
- Builds one image with all 7 apps baked in, `apps.json` passed as a BuildKit
  secret (not a build-arg, so it never leaks into image layer history)
- Pushes to Docker Hub: `amirul123/fortuntech-erpnext:latest`

Expect 20-35 minutes build time on GitHub's free runners (7 apps means 7x
`bench build` asset compilation).

## After the image is built

Point your `docker-compose.yml` at the new image:

```yaml
services:
  backend:
    image: amirul123/fortuntech-erpnext:latest
  # same for websocket, queue-long, queue-short, scheduler containers
```

Then register the apps against your site (fast — just DB registration, no
re-fetching code):

```bash
bench --site yoursite.domain install-app hrms crm helpdesk insights builder myinvois
```

## Local test build (before pushing to GitHub, optional)

```bash
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
docker build \
  --build-arg=FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg=FRAPPE_BRANCH=version-16 \
  --build-arg=PYTHON_VERSION=3.11 \
  --build-arg=NODE_VERSION=20 \
  --secret=id=apps_json,src=/path/to/apps.json \
  --tag=fortuntech-erpnext:test \
  --file=images/layered/Containerfile .
```

Requires Docker Engine v23.0+ (BuildKit default). Verify apps landed inside:

```bash
docker run -it fortuntech-erpnext:test /bin/bash
ls apps/
```
