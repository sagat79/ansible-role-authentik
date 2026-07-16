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
- **LDAP and RADIUS outposts** — dedicated outpost containers for the providers that the embedded outpost cannot serve. See the `outposts` section in [`defaults/main.yml`](defaults/main.yml). Proxy / forward-auth use cases are covered by the embedded outpost and do not need a dedicated container.
- **S3 storage** — store uploaded files (media, reports) in an S3-compatible bucket via `authentik_storage_backend: s3`.
- **Prometheus metrics** — enable `authentik_metrics_enabled` to expose metrics via Traefik. Since authentik 2025.8, metrics (port 9300) are served by the **worker** container, so the metrics labels are attached to it.
- **Tuning knobs** — PostgreSQL connection options, gunicorn workers/threads, background worker processes/threads, cache/session timeouts, trusted proxy CIDRs, GeoIP databases and more.

### Note for IPv4-only hosts

Since authentik 2026.5, the server listens on `[::]` by default. If your container network has no IPv6, set:

```yaml
authentik_listen_http: 0.0.0.0:9000
```

## Development

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit. See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```
