<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Changelog

## 2026-07-15

### authentik upgraded to 2026.5.2 (potentially breaking)

The role was synchronized with the upstream
[mother-of-all-self-hosting/ansible-role-authentik](https://github.com/mother-of-all-self-hosting/ansible-role-authentik)
role, upgrading authentik from 2025.4.1 to 2026.5.2. If you are upgrading an
existing installation, note authentik's own breaking changes along the way:

- **2025.8**: background tasks moved from Celery/Redis to PostgreSQL.
  High-traffic instances should consult the
  [2025.8 release notes](https://docs.goauthentik.io/releases/2025.8) before upgrading,
  as in-flight tasks may be lost.
- **2025.10**: Redis was removed entirely. All `authentik_config_redis_*`
  variables are gone (the role's configuration validation will point you at
  any leftovers). Expect ~50% more PostgreSQL connections.
- **2025.12+**: uploaded files moved from `/media` to `/data`. The role
  migrates the old directory automatically.
- **2026.5**: the server listens on `[::]` by default. On IPv4-only container
  networks, set `authentik_listen_http: 0.0.0.0:9000`.

authentik supports upgrading sequentially through major versions. Coming from
2025.4, upgrade through 2025.6 → 2025.8 → 2025.10 → 2026.2 → 2026.5 if you
want to follow the officially supported path (set `authentik_version`
accordingly for each run), or take a database backup before jumping directly.

Any dedicated outposts must run the same version as the authentik instance;
the role's outpost images follow `authentik_version` automatically.

### Path prefix serving changed (breaking, only if you serve under a subpath)

When `authentik_path_prefix` is not `/`, the role now configures authentik
itself for subpath serving (`AUTHENTIK_WEB__PATH`) and no longer strips the
prefix at the Traefik level. If you had overridden
`authentik_container_labels_traefik_path_prefix` separately from
`authentik_path_prefix`, the configuration validation will now ask you to
align them.

### Worker networking variables now take effect

`authentik_worker_container_network` and
`authentik_worker_container_additional_networks` were previously defined but
unused. They are now honored; the additional-networks list defaults to the
server's list, so existing setups keep working unchanged.

### Metrics are served by the worker

Since authentik 2025.8, Prometheus metrics (port 9300) are served by the
worker container. The metrics Traefik labels are now attached to the worker
(via a new `labels.worker` file) instead of the server. Optional HTTP Basic
Auth protection is available via
`authentik_metrics_container_labels_traefik_basicauth_*`.

### New features

- LDAP and RADIUS outposts as optional systemd services
  (`authentik_outpost_ldap_*`, `authentik_outpost_radius_*`).
- Blueprints support (`authentik_blueprints`) for declarative authentik
  configuration.
- Bootstrap variables (`authentik_bootstrap_*`) for automating the initial
  admin setup.
- S3 storage backend (`authentik_storage_*`).
- Many new first-class configuration variables: PostgreSQL tuning, web/worker
  process tuning, trusted proxy CIDRs, cookie domain, cache/session/reputation
  timeouts, GeoIP databases, listen addresses.
- Bug fixes: `authentik_loglevel` and `authentik_database_port` are now
  actually passed to the container; the SMTP timeout is configurable via
  `authentik_email_timeout`.
