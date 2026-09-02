# Nextcloud + ONLYOFFICE Docs + Traefik + Let's Encrypt — Docker Compose

[![Deployment Verification](https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml/badge.svg?branch=main)](https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

This repository deploys **Nextcloud** with a full **ONLYOFFICE Docs** document server — collaborative editing of Word, Excel, and PowerPoint files inside your own cloud — behind **Traefik** with automatic **Let's Encrypt TLS**. Nine services: Nextcloud with PostgreSQL 16 and Redis, ONLYOFFICE Docs with its own PostgreSQL, Redis, and RabbitMQ, Traefik, and a scheduled backup container with companion restore scripts.

📙 Full narrative installation guide on the blog: [heyvaldemar.com/install-nextcloud-with-onlyoffice-using-docker-compose/](https://www.heyvaldemar.com/install-nextcloud-with-onlyoffice-using-docker-compose/).

## Getting started

You need two DNS records pointing at this server: one for Nextcloud, one for the document server.

```bash
# 1. Clone
git clone https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose
cd nextcloud-onlyoffice-traefik-letsencrypt-docker-compose

# 2. Create the three Docker networks the stack expects
docker network create traefik-network
docker network create nextcloud-network
docker network create onlyoffice-network

# 3. Copy the environment template and fill in required values
cp .env.example .env
$EDITOR .env
# ^ Required: both hostnames, NEXTCLOUD_URL, five generated secrets,
#   ONLYOFFICE_DOCUMENT_JWT_SECRET, TRAEFIK_ACME_EMAIL, TRAEFIK_BASIC_AUTH.

# 4. Deploy
docker compose -f nextcloud-onlyoffice-traefik-letsencrypt-docker-compose.yml -p nextcloud up -d
```

Nextcloud installs itself on first start (admin account from `.env`). Once both services answer, connect them: in Nextcloud, install the **ONLYOFFICE** app from the app store, then in **Administration settings → ONLYOFFICE** set the Document Server address to `https://your-onlyoffice-hostname` and paste your `ONLYOFFICE_DOCUMENT_JWT_SECRET`.

### What success looks like

```bash
docker compose -f nextcloud-onlyoffice-traefik-letsencrypt-docker-compose.yml -p nextcloud ps
curl -fsk "https://${NEXTCLOUD_HOSTNAME}/status.php"   # {"installed":true,...}
curl -fsk "https://${ONLYOFFICE_DOCUMENT_HOSTNAME}/healthcheck"   # true
```

### Common first-deploy issues

- **Cert issuance fails.** DNS hasn't propagated for one of the two hostnames, or port 80 isn't reachable.
- **`docker compose up` fails with `set in .env`.** A required variable is empty; the error names it.
- **Networks not found.** Step 2 was skipped — all three are required.
- **ONLYOFFICE says \"download failed\" when opening a document.** The two services talk to each other server-side: both hostnames must resolve from inside the containers (public DNS, not just your laptop's hosts file), and the JWT secret in the connector app must match `.env`.

## Supply chain trust

Eight images — [`traefik`](https://hub.docker.com/_/traefik), [`nextcloud`](https://hub.docker.com/_/nextcloud), [`postgres`](https://hub.docker.com/_/postgres) ×2, [`redis`](https://hub.docker.com/_/redis) ×2, [`onlyoffice/documentserver`](https://hub.docker.com/r/onlyoffice/documentserver), [`rabbitmq`](https://hub.docker.com/_/rabbitmq) — pinned to `tag@sha256:<digest>` as interpolation defaults in the compose `x-images` block. `git pull` alone delivers the tested combination; an `*_IMAGE_TAG` variable in `.env` overrides deliberately.

The weekly `check-pin-freshness` CI job re-resolves each pin against its registry and compares the pinned Nextcloud, ONLYOFFICE, and Traefik versions against the latest upstream releases. GitHub Actions are pinned by commit SHA; Dependabot keeps those fresh.

## Production checklist

- [ ] **Strong secrets** — five generated passwords plus the JWT secret, 24+ random characters each.
- [ ] **Both DNS records** in place before first start, so Let's Encrypt issues on the first attempt.
- [ ] **Verify the ONLYOFFICE connection** by opening a document — it exercises the JWT secret and server-side connectivity in both directions.
- [ ] **Host-mount the backup volumes** for disaster recovery.
- [ ] **Upgrade Nextcloud one major at a time** — never skip majors; see the release notes.

## Backups and restore

The `backups` container runs a `pg_dump | gzip` + `tar.gz`-of-data → prune → sleep loop against the Nextcloud database (defaults: 30-minute warm-up, 24-hour interval, 7-day retention). Restore with the interactive scripts (`chmod +x *.sh` once): `./nextcloud-restore-database.sh`, then `./nextcloud-restore-application-data.sh`.

Worth knowing: before v1.0.0 the backup loop pointed at a database host that does not exist in this stack, so it never produced a single backup. If you deployed an earlier revision, check that `/srv/nextcloud-postgres/backups` actually has files dated after your upgrade.

## Testing

The [Deployment Verification](https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose/actions/workflows/deployment-verification.yml?query=branch%3Amain) workflow runs on every push, pull request, and every Monday at 06:00 UTC: shellcheck + actionlint, Trivy scans of the pinned images, the weekly freshness check, and a deploy-and-test job that boots all nine services with ephemeral credentials and requires Nextcloud's `status.php` to report `installed:true` and the ONLYOFFICE `/healthcheck` to return `true`, both through Traefik.

### Backup and restore, proven

`tests/e2e-backup-restore.sh` runs against the live stack and is what CI executes after the HTTPS smoke. The scenario that matters most is the restore roundtrip: insert a marker row, restore the earliest backup, assert the marker is gone — a backup that cannot be restored fails the build. Run it yourself against a running deployment with short intervals in `.env` (`BACKUP_INIT_SLEEP=15s`, `BACKUP_INTERVAL=60s`):

```bash
chmod +x tests/e2e-backup-restore.sh
./tests/e2e-backup-restore.sh
```

It stops the database container briefly to prove failure detection — run it on a staging copy, not on production.

## Security Notes

- Credentials are read from `.env` at deploy time; `.env` is gitignored and compose fails fast on missing required variables.
- **Pre-rotation advisory.** Releases before v1.0.0 (2026-08-31) shipped a tracked `.env` with generated-looking passwords for the database, Redis, the Nextcloud admin, ONLYOFFICE, and RabbitMQ. Rotate them all if your deployment reused them.
- JWT auth between Nextcloud and ONLYOFFICE is enabled by default (`JWT_ENABLED: true`) — without it, anyone who can reach the document server can feed it documents.
- Databases, Redis, and RabbitMQ listen only on internal networks.

---

## About the maintainer

<div align="center">

**Maintained by [Vladimir Mikhalev](https://github.com/heyvaldemar)** — Docker Captain · IBM Champion · AWS Community Builder

[YouTube](https://www.youtube.com/channel/UCf85kQ0u1sYTTTyKVpxrlyQ?sub_confirmation=1) · [Blog](https://heyvaldemar.com) · [LinkedIn](https://www.linkedin.com/in/heyvaldemar/)

</div>
