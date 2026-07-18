<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Testing this role on a real server

This guide describes how to validate this role end-to-end on a dedicated test
machine (e.g. a Proxmox VM), using the [MASH playbook](https://github.com/mother-of-all-self-hosting/mash-playbook)
for all the wiring (Docker, PostgreSQL, Traefik).

## Test machine

- Ubuntu 24.04 / Debian 12+ VM: 4 vCPU (host CPU type), 6–8 GB RAM, 32+ GB disk
- a user with sudo and your SSH key; `python3` installed
- take a VM snapshot (`clean-os`) before the first run, and roll back to it
  between destructive tests
- DNS: point a test hostname (e.g. `sso-test.example.com`) at the machine.
  For the Brands test, add a second hostname (e.g. `sso-test2.example.com`).

## Playbook setup

```sh
git clone https://github.com/mother-of-all-self-hosting/mash-playbook.git
cd mash-playbook
```

Edit `requirements.yml` so the authentik role comes from this repository and
branch instead of the pinned upstream release:

```yaml
- src: git+https://github.com/sagat79/ansible-role-authentik.git
  version: claude/authentik-mash-role-plan-zme0d4
  name: authentik
  activation_prefix: authentik_
```

Then fetch the roles (`just roles` or `make roles`, depending on the playbook
version) and create the inventory:

`inventory/hosts`:

```ini
[mash_servers]
# mash.example.com — the name Ansible uses for the machine (any name works);
# ansible_host — the VM's actual IP; ansible_ssh_user — the cloud-init user
# you SSH in as; become=true — use sudo for privileged tasks.
mash.example.com ansible_host=<VM-IP> ansible_ssh_user=<user> become=true
```

`inventory/host_vars/mash.example.com/vars.yml` (minimal test config; every
variable is explained in the comment above it):

```yaml
---
########################################################################
# General playbook settings
########################################################################

# The playbook's main "family" key. The playbook automatically derives
# various internal passwords from it (for example, the password authentik
# uses to connect to Postgres) — so you don't have to invent a password for
# every service yourself. Generate it once and NEVER change it afterwards
# (otherwise the derived passwords no longer match what's already in the
# database).
# Generate with: pwgen -s 64 1   (or: openssl rand -hex 32)
mash_playbook_generic_secret_key: ''

# Tells the playbook to install Docker on the server itself.
# Leave it true — don't install Docker by hand, so everything stays uniform.
mash_playbook_docker_installation_enabled: true

# Installs the Python library Ansible uses to talk to Docker
# (without it, the image/network tasks fail). Just leave it true.
devture_docker_sdk_for_python_installation_enabled: true

########################################################################
# Traefik (the reverse proxy — it terminates HTTPS and routes to containers)
########################################################################

# Enables Traefik. It sits "in front", obtains certificates from
# Let's Encrypt and forwards requests to authentik over the internal
# Docker network.
traefik_enabled: true

# The email Traefik registers with at Let's Encrypt.
# You'll get a warning there if a certificate is about to expire and fails
# to renew. Use a real address; it doesn't have to be on the same domain.
traefik_config_certificatesResolvers_acme_email: you@example.com

# Note: by default Let's Encrypt validates the domain by connecting to
# ports 80/443 on the machine (HTTP challenge) — that requires a port
# forward on your router. If the test lives only on your home LAN, use a
# DNS-01 challenge instead (see the playbook's Traefik documentation — it
# needs an API token for your DNS provider), or accept a self-signed
# certificate.

########################################################################
# PostgreSQL (authentik's database)
########################################################################

# Enables the playbook's Postgres container. authentik is pointed at it
# automatically — you don't configure anything else for the connection.
postgres_enabled: true

# The Postgres superuser password. It's only used internally by the
# playbook (you never type it anywhere yourself). Generate it and forget it.
# Generate with: pwgen -s 64 1
postgres_connection_password: ''

########################################################################
# authentik
########################################################################

# Enables authentik itself (the server + worker containers).
authentik_enabled: true

# The domain you'll open authentik at in the browser. You need a DNS record
# (or an /etc/hosts line on your laptop) pointing it at the VM's IP.
authentik_hostname: sso-test.example.com

# The secret key authentik signs session cookies with.
# Generate once; changing it later logs out all users.
# Generate with: pwgen -s 64 1
authentik_secret_key: ''

# --- Initial administrator (bootstrap) ---
# These three lines create the "akadmin" admin account automatically on the
# FIRST start, so you don't have to click through the setup screen in the
# browser. They only take effect on a fresh installation — changing them
# later has no effect.

# The admin account's email (you log in with it).
authentik_bootstrap_email: admin@example.com

# The admin account's password. This is a test environment — keep it
# simple, but do NOT reuse one of your real passwords.
authentik_bootstrap_password: ''

# Optional: a ready-made API token for the admin. Handy for the checks
# below (curl against the API without a browser login).
# Generate with: pwgen -s 48 1
authentik_bootstrap_token: ''

# --- Metrics (for test 3) ---
# Publishes authentik's Prometheus metrics via Traefik at
# https://<hostname>/metrics. In a real environment you'd also add Basic
# Auth (authentik_metrics_container_labels_traefik_basicauth_*); for the
# test it's not needed.
authentik_metrics_enabled: true

# --- Brands (for test 6) — uncomment AFTER the base install works ---
# Brands = a different look (title, logo) depending on the domain you come
# from. The first block tells Traefik to also accept the second domain
# (otherwise requests for it never reach authentik at all). It needs a DNS
# record too!
# authentik_container_labels_traefik_additional_hostnames:
#   - sso-test2.example.com
#
# The brand list: domain = which domain gets which look;
# branding_title = the title shown in the UI; default: true = the fallback
# brand used when no domain matches (only one entry may have it).
# authentik_brands:
#   - domain: sso-test.example.com
#     branding_title: Test SSO
#   - domain: sso-test2.example.com
#     branding_title: Second Brand
#     default: true
```

Install:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=install-all,start
```

## Variant B: on a server that also runs matrix-docker-ansible-deploy

If the test machine (or the eventual production server) already runs the
[matrix-docker-ansible-deploy](https://github.com/spantaleev/matrix-docker-ansible-deploy)
playbook, its Traefik should keep managing HTTPS, and MASH must be told to
reuse it instead of starting its own. This mirrors the upstream example
[`examples/mash-for-matrix-docker-ansible-deploy-users/vars.yml`](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/examples/mash-for-matrix-docker-ansible-deploy-users/vars.yml).

Replace the Traefik section of the `vars.yml` above with:

```yaml
# Prefix all MASH services (mash-authentik, mash-postgres, ...) so their
# container/service names never collide with the matrix-* ones.
mash_playbook_service_identifier_prefix: 'mash-'
mash_playbook_service_base_directory_name_prefix: 'mash-'

# Don't install a MASH-managed Traefik — the Matrix playbook already runs one.
mash_playbook_reverse_proxy_type: other-traefik-container

# The name of the Docker network the Matrix playbook's Traefik lives on.
# MASH services (authentik included) get attached to it so Traefik can
# reach them. `traefik` is the Matrix playbook's default network name.
mash_playbook_reverse_proxyable_services_additional_network: traefik

# The Matrix playbook's Traefik listens on the `web-secure` entrypoint and
# uses the `default` certificate resolver — which are exactly this role's
# defaults, so no extra authentik_* Traefik settings are needed.
```

Everything else (Postgres, authentik, the tests below) stays the same. Note
that with this variant the systemd services are named `mash-authentik-server`
/ `mash-authentik-worker` (because of the prefix), so adjust the `systemctl`
commands in the checklist accordingly.

## Test checklist

1. **Fresh install + bootstrap**: `systemctl status mash-authentik-server
   mash-authentik-worker` are active; `docker ps` shows both containers
   healthy. Log in at `https://sso-test.example.com` with the bootstrap
   credentials — the initial-setup flow should NOT be required.
2. **Worker health**: `docker exec mash-authentik-worker ak healthcheck` (or
   check the container's health status).
3. **Metrics**: `curl -H 'Host: sso-test.example.com'
   https://<VM-IP>/metrics` returns Prometheus metrics (served by the
   *worker* container). If you enabled
   `authentik_metrics_container_labels_traefik_basicauth_*`, verify a 401
   without credentials and a 200 with them.
4. **Blueprints**: add a simple entry to `authentik_blueprints_custom`,
   re-run the playbook, and confirm the blueprint appears (and applies) under
   **Customization → Blueprints** in the admin UI.
5. **LDAP outpost**: create an LDAP provider + outpost in the admin UI, copy
   the token into `authentik_outpost_ldap_token`, set
   `authentik_outpost_ldap_enabled: true` and
   `authentik_outpost_ldap_container_ldap_host_bind_port: "389"`, re-run,
   then test with `ldapsearch -H ldap://<VM-IP> -D
   "cn=<user>,ou=users,dc=ldap,dc=goauthentik,dc=io" -w <password>`.
   Change the token and re-run to confirm
   `authentik_outposts_restart_necessary` triggers a restart.
6. **Brands**: uncomment the brands block above, re-run, then open both
   hostnames — each should show its own title/branding, and the second one is
   the default (fallback) brand. Check the TLS certificate covers both names.
7. **Uninstall**: set `authentik_enabled: false`, re-run with
   `--tags=setup-all`, and confirm all authentik services and containers
   (including outposts) are gone.

Roll back to the `clean-os` snapshot and repeat from step 1 for a full
regression pass after role changes.

## Multi-service test plan: authentik + Ghost on one host

This section extends the guide into a phased plan that also installs and
tests the [Ghost](https://ghost.org/) blogging platform via the
[derfeldev/ansible-role-ghost](https://github.com/derfeldev/ansible-role-ghost)
role, alongside authentik.

### Domains (all pointing at the test machine's IP)

| Domain | Service | Phase |
| --- | --- | --- |
| `sso-test.example.com` | authentik (main hostname) | 2 |
| `sso-test2.example.com` | authentik — second brand (Brands test) | 4 |
| `blog-test.example.com` | Ghost | 3 |

The mailer (exim-relay) and the databases are internal-only and need no
public domains.

### Additional requirements.yml entry

Next to the authentik entry from the top of this guide, also add:

```yaml
- src: git+https://github.com/derfeldev/ansible-role-ghost.git
  version: main
  name: ghost
  activation_prefix: ghost_
```

### Additional vars.yml sections (Ghost + its database + mail)

```yaml
########################################################################
# Mailer (exim-relay) — outgoing email for all services
########################################################################

# A small SMTP relay all services send mail through. For a LAN test the
# mails will likely land in spam (no reverse DNS / DKIM) — that's fine,
# we only verify the sending path works.
exim_relay_enabled: true
exim_relay_hostname: mail.example.com
exim_relay_sender_address: test@example.com

########################################################################
# MariaDB (Ghost's database)
########################################################################

# Ghost officially supports MySQL 8 only. The MASH playbook ships a MariaDB
# role, which works for most installs when Ghost is told to speak the mysql
# protocol — acceptable for a test environment, but note the caveats in the
# Ghost role's own documentation (JSON columns, full-text search edge cases)
# before using this in production.
mariadb_enabled: true
# Generate with: pwgen -s 64 1
mariadb_root_passphrase: ''

########################################################################
# Ghost
########################################################################

# Enables Ghost itself.
ghost_enabled: true

# The domain the blog is served at (needs a DNS record, like the others).
ghost_hostname: blog-test.example.com

# Database connection — point Ghost at the playbook-managed MariaDB.
ghost_database_hostname: "{{ mariadb_connection_hostname }}"
ghost_database_username: ghost
# Generate with: pwgen -s 64 1
ghost_database_password: ''

# Mail through the exim-relay above (see the Ghost role's
# docs/mash-playbook-integration.md for the full option list).
ghost_mail_enabled: true
ghost_mail_options_host: "{{ exim_relay_identifier }}"
ghost_mail_options_port: 8025
ghost_mail_options_secure: false
ghost_mail_from: test@example.com
```

Also register the MariaDB database for Ghost the way your playbook version
expects (`mariadb_managed_databases` list entry with the name/username/
password above), and make sure Ghost's container joins the MariaDB and
exim-relay networks if your playbook version doesn't wire that
automatically.

### Phased execution

Each phase must end green before moving on:

- **Phase 0 — prep**: DNS records; VM snapshot (`clean-os`).
- **Phase 1 — base**: Traefik + Postgres + exim-relay. Check: services
  active, Traefik answers on 443.
- **Phase 2 — authentik**: install + checklist tests 1–4 above (bootstrap
  login, worker health, metrics, blueprint).
- **Phase 3 — Ghost**: MariaDB + the Ghost role. Check:
  `https://blog-test.example.com` serves the blog; `/ghost` admin setup
  completes; a test email goes out through exim-relay; create a post,
  restart the Ghost container, confirm the post persists.
- **Phase 4 — extensions**: authentik Brands (both domains, different
  branding) + the LDAP outpost (checklist tests 5–6).
- **Phase 5 — destructive**: `authentik_enabled: false` and
  `ghost_enabled: false` → clean removal (checklist test 7); roll back to
  the snapshot and repeat the full cycle for an idempotency pass.
