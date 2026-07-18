<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Improvement plan for ansible-role-authentik

Date: 2026-07-15

## Context and findings

This repository is a fork of the MASH authentik role, frozen at **authentik 2025.4.1**. In the meantime:

- The official MASH role ([mother-of-all-self-hosting/ansible-role-authentik](https://github.com/mother-of-all-self-hosting/ansible-role-authentik)) is at **v2026.5.2-0**, and exactly that version is pinned in the mash-playbook's `templates/requirements.yml`.
- authentik went through several incompatible changes:
  - **2025.8** — tasks moved from Celery/Redis to PostgreSQL (Dramatiq); `AUTHENTIK_BROKER__*` and `AUTHENTIK_RESULT_BACKEND__*` were removed; `AUTHENTIK_WORKER__CONCURRENCY` → `AUTHENTIK_WORKER__THREADS`; metrics are served by the **worker** container, not the server.
  - **2025.10** — **Redis was removed entirely** (cache, WebSocket and the embedded outpost move to Postgres; ~50% more database connections).
  - **2025.12+** — storage rework: `/media` → `/data` (`AUTHENTIK_STORAGE__*`).
  - **2026.5** — Rust worker entrypoint (−200 MB memory), default listen address `[::]` instead of `0.0.0.0`, new `AUTHENTIK_BOOTSTRAP_PASSWORD_HASH`.
- Upgrades must go **sequentially through the major versions** (2025.4 → 2025.6 → 2025.8 → 2025.10 → 2026.2 → 2026.5); 2025.8 needs care on busy instances (task queue migration). Outpost versions must be switched together with the instance.

---

## Stage 0 — Sync with upstream (the biggest win)

Recommendation: instead of rewriting the same work, **rebase the fork onto upstream v2026.5.2-0** and layer the local improvements on top. The local commits (renovate.json, autotag.yml, small fixes) already exist upstream in equivalent form.

What the sync brings (differences from the current state):

1. **authentik 2025.4.1 → 2026.5.2** (+ CHANGELOG note about the sequential upgrade path and the 2025.8 task migration).
2. **Redis removal**: `authentik_config_redis_*`, `CELERY_BROKER_URL` and `AUTHENTIK_REDIS__*` gone from the env template; deprecation checks in `validate_config.yml` for all old names.
3. **Worker as a standalone container** (`… server:TAG worker`) with its own `docker create` — instead of the fragile `docker exec` into the server container. Own healthcheck, networks, hardening (`--cap-drop=ALL`, `--read-only`, tmpfs), pre-start cleanup, `Requires`/`Wants` lists.
4. **`/media` → `/data`** + a migration task that moves the old directory.
5. **`AUTHENTIK_POSTGRESQL__SSLMODE`** is actually passed (currently `authentik_database_sslmode` is defined but never reaches env — a bug).
6. **Path prefix support**: `authentik_path_prefix` + Traefik `PathPrefix`/strip-prefix/slashless-redirect middlewares.
7. Modern MASH conventions:
   - the `authentik_container_image_registry_prefix_upstream(_default)` chain (for mirror registries);
   - `authentik_container_extra_arguments(_auto/_custom)`;
   - `authentik_container_labels_additional_labels_auto/_custom` (a list instead of a string);
   - `authentik_restart_necessary` (computed from config/image changes) — required by the mash-playbook wiring;
   - image pulls via `community.docker.docker_image_pull` or the `command` method;
   - networks via `community.docker.docker_network` (not `community.general`);
   - `authentik_container_tmpfs_tmp_size`, stop-grace/cleanup timeouts;
   - `TZ`, `AUTHENTIK_ERROR_REPORTING__ENABLED` variables;
   - `X-Frame-Options` dropped (duplicates CSP `frame-ancestors`), `Permission-Policy` → `Permissions-Policy` (spelling bug);
   - typo fix `authentik_custon_templates_path` → `authentik_custom_templates_path`.
8. **Tooling**: REUSE/SPDX headers, `.pre-commit-config.yaml`, `.yamllint.yml`, `.ansible-lint`, `justfile`, `mise.toml`, an extended README, a fixed `meta/main.yml` (the current description was copied from a music service).

Steps:

- `git remote add upstream …/mother-of-all-self-hosting/ansible-role-authentik`
- new branch from `upstream/main`; carry over any local specifics;
- tag following the MASH convention `v2026.5.2-0` (autotag is already present).

---

## Stage 1 — Fix bugs that also exist upstream (fixed in this repo)

1. **`authentik_loglevel` is a dead variable** — defined in defaults, but `AUTHENTIK_LOG_LEVEL` is never written to env. Add it to `env.j2`.
2. **`authentik_database_port` is not passed** — `AUTHENTIK_POSTGRESQL__PORT` is missing from the env template (works only because 5432 is the default).
3. **Metrics are on the wrong container** — since 2025.8, port 9300 is served by the worker, while the metrics Traefik labels sit on the server container. Solution: a separate labels file for the worker.
4. **`AUTHENTIK_EMAIL__TIMEOUT` is hardcoded to 10** — make it a variable (`authentik_email_timeout`).
5. **IPv4-only environments**: since 2026.5 the default listen address is `[::]` — add a variable for `AUTHENTIK_LISTEN__HTTP` and a README note.

---

## Stage 2 — Full coverage of authentik's configuration

First-class variables (with sensible defaults; everything else remains possible via `authentik_environment_variables_additional_variables`):

| Group | Variables |
| --- | --- |
| PostgreSQL | `PORT`, `CONN_MAX_AGE`, `CONN_HEALTH_CHECKS`, `DISABLE_SERVER_SIDE_CURSORS`, `DEFAULT_SCHEMA`, read replicas |
| Web | `WEB__WORKERS`, `WEB__THREADS`, `WEB__PATH` (**required when `authentik_path_prefix != '/'`** — set automatically!) |
| Worker | `WORKER__PROCESSES`, `WORKER__THREADS` |
| Listen | `LISTEN__TRUSTED_PROXY_CIDRS` (important behind Traefik for correct client IPs) |
| Storage | `STORAGE__BACKEND` file/s3 + the full S3 set (endpoint, region, bucket, keys, custom domain) — media on S3 |
| Bootstrap | `AUTHENTIK_BOOTSTRAP_EMAIL/PASSWORD/TOKEN` + `PASSWORD_HASH` (2026.5) — automated initial admin setup without the manual initial-setup flow |
| Sessions/cache | `SESSIONS__UNAUTHENTICATED_AGE`, `CACHE__TIMEOUT*`, `REPUTATION__EXPIRY` |
| Misc | `COOKIE_DOMAIN`, `DISABLE_UPDATE_CHECK`, `EMAIL__TIMEOUT`, `LOG_LEVEL` |
| GeoIP | `EVENTS__CONTEXT_PROCESSORS__GEOIP/ASN` + optional bind mount of MaxMind databases from the host |
| Outposts | `OUTPOSTS__CONTAINER_IMAGE_BASE`, `OUTPOSTS__DISCOVER` |

---

## Stage 3 — New capabilities beyond upstream

1. **Outpost sub-services** (the biggest added value; the MASH docs explicitly say LDAP is untested):
   - LDAP outpost (`ghcr.io/goauthentik/ldap`) — a separate systemd service, `AUTHENTIK_HOST/TOKEN/INSECURE`, ports 3389/6636;
   - RADIUS outpost (`ghcr.io/goauthentik/radius`);
   - Proxy outpost (`ghcr.io/goauthentik/proxy`) for forward auth of services beyond the embedded outpost;
   - general note: outpost version = instance version (tied to `authentik_version`).
2. **Blueprints** — mount `/blueprints` + let the playbook supply declarative YAML blueprint files (flows, applications, providers) → reproducible configuration without clicking through the UI.
3. **Monitoring wiring** — integration with the `mash_playbook_metrics_exposure_*` pattern (Prometheus scrape of worker:9300, optional basic auth protection).
4. **`AUTHENTIK_LISTEN__METRICS` / debug ports** — disable/restrict.

---

## Stage 3.5 — Brands (visual settings and defaults per domain)

authentik Brands allow different branding (title, logo, favicon, custom CSS) and different default flows for each domain the instance is accessed under (matched on the `Host` header).

1. **Review** of what the role supports today:
   - the Traefik router only matches `authentik_hostname` — requests for additional brand domains never reach the container at all;
   - `authentik_cookie_domain` is a single value — document the behavior with several unrelated domains;
   - brand logos/favicons live in media storage (`/data`, already supported, incl. S3) — check what paths authentik expects for uploads via blueprint vs. the UI.
2. **Correction**:
   - a new variable `authentik_container_labels_traefik_additional_hostnames` (list) — extends the router rule to ``Host(`a`) || Host(`b`) …`` so brand domains get served;
   - declarative brands: an `authentik_brands` list (domain, branding_title, branding_logo, branding_favicon, default flag), from which the role generates a blueprint file (`model: authentik_brands.brand`) via the existing blueprints mechanism — no UI clicking;
   - validation: at most one default brand; a warning for a brand domain missing from the Traefik hostnames list;
   - a README section with a second-domain example.
3. **Test**:
   - render tests: a router rule with multiple domains; the generated brands blueprint (valid YAML, correct identifiers/attrs); a negative test for two default brands;
   - a real test on the test server: second domain → different logo/title; does the main domain fall back to the default brand; TLS certificate for the additional domain (does the certResolver cover both hostnames).

---

## Stage 4 — Documentation (this repository only)

1. Role README following the MASH standard + CHANGELOG entries for all breaking changes (Redis, /data, upgrade path).
2. README examples for use with the MASH playbook: forward auth via Traefik, LDAP/RADIUS outposts, bootstrap variables. All changes stay in this repository — no PRs to mash-playbook or the upstream role are planned.

---

## Stage 5 — Quality

- ansible-lint + yamllint + pre-commit (arrive with Stage 0), Woodpecker CI;
- REUSE compliance (SPDX headers everywhere);
- optional: a molecule scenario for a smoke test (server + worker start, healthcheck green);
- Renovate keeps `authentik_version` current (already configured).

## Recommended execution order

1. Stage 0 (sync) — immediately; everything else builds on it. ✅
2. Stage 1 (bugs) — small and quick fixes. ✅
3. Stage 2 + Stage 3.1 (outposts) — the main new functionality. ✅
4. Stages 3.2–3.4 (blueprints, monitoring, listen) and Stage 4 (documentation). ✅
5. Stage 3.5 (Brands) — review, correction and test; the real test is combined with validating the role on the home test server.
6. Stage 5 (quality) — the molecule scenario remains open.
