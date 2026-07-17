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
mash.example.com ansible_host=<VM-IP> ansible_ssh_user=<user> become=true
```

`inventory/host_vars/mash.example.com/vars.yml` (minimal test config):

```yaml
---
mash_playbook_generic_secret_key: ''  # pwgen -s 64 1

# Docker + Traefik + Postgres come from the playbook
mash_playbook_docker_installation_enabled: true
devture_docker_sdk_for_python_installation_enabled: true

traefik_enabled: true
traefik_config_certificatesResolvers_acme_email: you@example.com
# For a LAN-only test without port forwarding, prefer a DNS-01 challenge
# (see the playbook's Traefik documentation), or use self-signed certificates.

postgres_enabled: true
postgres_connection_password: ''  # pwgen -s 64 1

# authentik
authentik_enabled: true
authentik_hostname: sso-test.example.com
authentik_secret_key: ''  # pwgen -s 64 1

# Skip the manual initial-setup flow:
authentik_bootstrap_email: admin@example.com
authentik_bootstrap_password: ''  # test-only password
authentik_bootstrap_token: ''     # optional API token, handy for the checks below

# Metrics (test 3):
authentik_metrics_enabled: true

# Brands (test 6) — uncomment after the base install works:
# authentik_container_labels_traefik_additional_hostnames:
#   - sso-test2.example.com
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
