<!--
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Slavi Pantaleev
SPDX-FileCopyrightText: 2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# authentik Ansible role

This is an [Ansible](https://www.ansible.com/) role which installs [authentik](https://goauthentik.io/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Features

Besides installing the authentik server and worker containers, this role supports:

- **Bootstrap of the initial admin user** — set `authentik_bootstrap_email` and `authentik_bootstrap_password` (or, better, `authentik_bootstrap_password_hash`) to skip the manual `/if/flow/initial-setup/` flow on a fresh installation.
- **LDAP and RADIUS outposts** — dedicated outpost containers for the providers that the embedded outpost cannot serve. Proxy / forward-auth use cases are covered by the embedded outpost and do not need a dedicated container.
- **Blueprints** — declarative configuration of flows, applications, providers etc. from YAML, applied automatically by the worker.
- **Brands** — per-domain visual settings and defaults (title, logo, favicon), configured declaratively and routed via additional Traefik hostnames.
- **S3 storage** — store uploaded files (media, reports) in an S3-compatible bucket via `authentik_storage_backend: s3`.
- **Prometheus metrics** — expose metrics via Traefik, optionally protected with HTTP Basic Auth. Since authentik 2025.8, metrics (port 9300) are served by the **worker** container, so the metrics labels are attached to it.
- **Tuning knobs** — PostgreSQL connection options, gunicorn workers/threads, background worker processes/threads, cache/session timeouts, trusted proxy CIDRs, GeoIP databases and more.

## Usage examples

The examples below use `vars.yml` syntax as used by the MASH playbook. Check [`defaults/main.yml`](defaults/main.yml) for all options and their documentation.

### LDAP / RADIUS outpost

First create an outpost of the corresponding type in the authentik admin interface (**Applications → Outposts**) and copy its token (**View Deployment Info**). Then:

```yaml
authentik_outpost_ldap_enabled: true
authentik_outpost_ldap_token: YOUR_OUTPOST_TOKEN
# Optionally publish the LDAP/LDAPS ports on the host:
authentik_outpost_ldap_container_ldap_host_bind_port: "389"
authentik_outpost_ldap_container_ldaps_host_bind_port: "636"
```

If your playbook manages services via a systemd service manager role (like the MASH playbook does), also register the outpost service, e.g.:

```yaml
devture_systemd_service_manager_services_list_additional:
  - name: mash-authentik-outpost-ldap.service
    priority: 2500
    groups: [mash, authentik]
```

### Blueprints

```yaml
authentik_blueprints_custom:
  - name: example-app
    content: |
      version: 1
      metadata:
        name: Example application
      entries:
        - model: authentik_core.application
          identifiers:
            slug: example
          attrs:
            name: Example
```

### Brands (per-domain visual settings and defaults)

[Brands](https://docs.goauthentik.io/docs/sys-mgmt/brands) let authentik present different branding
(title, logo, favicon) and defaults depending on the domain it is accessed under.
The role applies them declaratively (via an auto-generated blueprint) and can route
the additional domains through Traefik:

```yaml
authentik_container_labels_traefik_additional_hostnames:
  - login.other-project.org

authentik_brands:
  - domain: sso.example.com
    branding_title: Example SSO
  - domain: login.other-project.org
    branding_title: Other Project
    branding_logo: /media/public/other-project.svg
```

Point the DNS records of the additional domains at the same server; when TLS is
enabled, the certificate resolver obtains certificates for them automatically.

### Prometheus metrics with Basic Auth

```yaml
authentik_metrics_enabled: true
authentik_metrics_container_labels_traefik_basicauth_enabled: true
# Generate with: htpasswd -nb prometheus SOME_PASSWORD
authentik_metrics_container_labels_traefik_basicauth_users: "prometheus:$apr1$..."
```

### Note for IPv4-only hosts

Since authentik 2026.5, the server listens on `[::]` by default. If your container network has no IPv6, set:

```yaml
authentik_listen_http: 0.0.0.0:9000
authentik_listen_metrics: 0.0.0.0:9300
```

## Development

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```
