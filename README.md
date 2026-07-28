# ERPNext All-in-One Custom Images

Custom Docker images built on `frappe_docker`, with official Frappe apps baked in
so `bench get-app` additions survive container recreation.

## Image variants

| Image tag | Frappe/ERPNext version | Apps included |
|---|---|---|
| `amirul123/erpnext-allinone:latest` | v16 | erpnext, hrms, crm, helpdesk, insights, builder, payments |
| `amirul123/erpnext-allinone:latest-myinvois` | v15 | same as above + MyInvois (LHDN e-Invoicing) |

**Why myinvois variant is pinned to v15**: ERPGulf's MyInvois app officially lists
"Supported versions: Version 15" on Frappe Cloud Marketplace. It may work on v16, but
that's untested by the vendor. If you want to try MyInvois on v16, change
`frappe_branch: version-15` to `version-16` for the `myinvois` entry in
`.github/workflows/build.yml` — but test thoroughly on staging first.

## One-time setup

1. Push this repo to `Amirul78800/erpnext-allinone` (or any name you like) on GitHub.
2. In repo Settings → Secrets and variables → Actions, add:
   - `DOCKERHUB_USERNAME` = `amirul123`
   - `DOCKERHUB_TOKEN` = a Docker Hub access token (Docker Hub → Account Settings →
     Security → New Access Token — do NOT use your account password)
3. Push to `main` or run the workflow manually (Actions tab → Build and Push →
   Run workflow).

## What the workflow does

- Checks out this repo (for `apps-official.json` / `apps-myinvois.json`)
- Checks out `frappe/frappe_docker` (upstream build source — it owns the Dockerfile)
- Builds two images in parallel (matrix), one per variant, each with its own
  `apps.json` baked in via `APPS_JSON_BASE64` build-arg
- Pushes both to Docker Hub under `amirul123/erpnext-allinone`

Build time: expect 15-30 minutes per variant on GitHub's free runners (7 apps is a
lot of `bench build` asset compilation). If it's too slow, ask about self-hosted
runners on your TrueNAS box instead of GitHub-hosted ones.

## Branch caveats to watch for during first build

- **CRM** uses branch `main` (not `version-16`) — this is correct, don't change it.
- **Helpdesk / Insights / Builder** only publish `main` and `develop` branches, no
  per-version branches. `main` is generally the current stable line, but if the
  build fails on one of these, check that app's GitHub repo for a recent branch
  rename before assuming your apps.json is wrong.
- **HRMS on v16** had a dependency-declaration bug in Jan 2026 (right after v16
  launched) that was fixed in later patch releases. If you're building against an
  old cached HRMS commit, bump to latest `version-16` HEAD.

## After the image is built

Point your existing `docker-compose.yml` (the one used for the ERPNext deployment
you already have) at the new image instead of the stock `frappe/erpnext:v16` image,
e.g.:

```yaml
services:
  backend:
    image: amirul123/erpnext-allinone:latest
  # ...same for websocket, queue-long, queue-short, scheduler containers
```

Then recreate the site's apps list — even though the code is baked in, you still
need to run, once per site:

```bash
bench --site yoursite.domain install-app hrms crm helpdesk insights builder payments
```

(add `myinvois` too if using that variant)

This step only registers the app against the site's database — it does NOT
re-fetch code, so it's fast and doesn't hit GitHub rate limits.

## Local test build (optional, before pushing to GitHub)

```bash
git clone https://github.com/frappe/frappe_docker
cd frappe_docker
export APPS_JSON_BASE64=$(base64 -w0 /path/to/apps-official.json)
docker build \
  --build-arg FRAPPE_PATH=https://github.com/frappe/frappe \
  --build-arg FRAPPE_BRANCH=version-16 \
  --build-arg APPS_JSON_BASE64=$APPS_JSON_BASE64 \
  --build-arg PYTHON_VERSION=3.11 \
  --build-arg NODE_VERSION=20 \
  --tag erpnext-allinone:test \
  --file images/custom/Dockerfile .
```
