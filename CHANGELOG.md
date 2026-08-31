# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_(no unreleased changes yet)_

## [1.0.0] - 2026-08-31

First semver release. Brings this template to the fleet standard established
in [keycloak-traefik-letsencrypt-docker-compose](https://github.com/heyvaldemar/keycloak-traefik-letsencrypt-docker-compose)
v1.2.0.

### Changed (BREAKING for existing deployments)

- **Nextcloud 29 (EOL) → 34.0.3.** Nextcloud upgrades exactly one major
  version at a time — ❗ existing deployments must step through
  30 → 31 → 32 → 33 → 34 via `NEXTCLOUD_IMAGE_TAG` overrides. See the
  release notes for the full procedure.
- **ONLYOFFICE Docs 8.1 → 9.4.0**, **Redis 7.2 → 7.4**,
  **RabbitMQ 4.0 → 4.3**, **Traefik 3.2 → 3.7** (3.2's Docker client
  cannot talk to Docker Engine 29). PostgreSQL stays on 16, now
  digest-pinned.
- **All images pinned by `tag@sha256:digest`** in the compose `x-images`
  block; `.env` now carries only secrets and deliberate overrides.

### Fixed

- **The scheduled database backup never ran**: the backup loop dumped
  from a host named `postgres`, but the service in this stack is
  `postgres-nextcloud`, so every `pg_dump` failed on DNS resolution.
  The restore script pointed at the same nonexistent host. Both now
  target `postgres-nextcloud`, and backup-loop variables are `$$`-escaped
  so the container shell resolves them at runtime.
- Shellcheck findings in both restore scripts.

### Security

- **Credentials untracked from git.** The tracked `.env` carried
  generated-looking passwords for the database, Redis, the Nextcloud
  admin, ONLYOFFICE, and RabbitMQ — rotate them all if reused.

### Added

- **Deployment Verification workflow**: shellcheck + actionlint; Trivy
  scans of all pinned images; weekly `check-pin-freshness` (digest drift
  + Nextcloud/ONLYOFFICE Docker Hub tag lag + Traefik release lag); and
  a deploy-and-test job that boots the full nine-service stack and
  requires Nextcloud's `status.php` to report `installed:true` and the
  ONLYOFFICE `/healthcheck` to return `true`, both through Traefik.

[Unreleased]: https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/heyvaldemar/nextcloud-onlyoffice-traefik-letsencrypt-docker-compose/releases/tag/v1.0.0
