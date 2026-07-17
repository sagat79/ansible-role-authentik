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
# mash.example.com — как Ansible нарича машината (произволно име);
# ansible_host — реалното IP на VM-а; ansible_ssh_user — потребителят от
# cloud-init, с който влизате по SSH; become=true — да ползва sudo.
mash.example.com ansible_host=<VM-IP> ansible_ssh_user=<user> become=true
```

`inventory/host_vars/mash.example.com/vars.yml` (minimal test config, всяка
променлива е обяснена в коментара над нея):

```yaml
---
########################################################################
# Общи настройки на playbook-а
########################################################################

# Главният "семеен" ключ на MASH playbook-а. От него playbook-ът си извежда
# автоматично разни вътрешни пароли (например паролата, с която authentik
# се връзва към Postgres) — така не се налага да измисляте парола за всяка
# услуга поотделно. Генерира се веднъж и НЕ се променя после (иначе
# изведените пароли ще се разминат с вече създадените в базата).
# Генериране: pwgen -s 64 1   (или: openssl rand -hex 32)
mash_playbook_generic_secret_key: ''

# Казва на playbook-а сам да инсталира Docker на сървъра.
# Оставете true — не инсталирайте Docker ръчно, за да е всичко еднообразно.
mash_playbook_docker_installation_enabled: true

# Инсталира Python библиотеката, с която Ansible управлява Docker
# (без нея задачите за образи/мрежи ще гърмят). Просто оставете true.
devture_docker_sdk_for_python_installation_enabled: true

########################################################################
# Traefik (reverse proxy — той поема HTTPS и насочва към контейнерите)
########################################################################

# Включва Traefik. Той стои "отпред", взима сертификати от Let's Encrypt
# и препраща заявките към authentik по вътрешната Docker мрежа.
traefik_enabled: true

# Имейлът, с който Traefik се представя пред Let's Encrypt.
# На него ще получите предупреждение, ако сертификат изтича и не се подновява.
# Слагайте реален имейл, не e нужно да е на същия домейн.
traefik_config_certificatesResolvers_acme_email: you@example.com

# Забележка: по подразбиране Let's Encrypt проверява домейна, като се свързва
# към порт 80/443 на машината (HTTP challenge) — това изисква port forward
# от рутера. Ако тестът е само в домашната мрежа, ползвайте DNS-01 challenge
# (описан в Traefik документацията на playbook-а — иска API token за DNS
# доставчика ви) или се примирете със self-signed сертификат.

########################################################################
# PostgreSQL (базата данни на authentik)
########################################################################

# Включва Postgres контейнера на playbook-а. authentik автоматично ще бъде
# насочен към него — нищо друго не настройвате за връзката.
postgres_enabled: true

# Паролата на superuser-а на Postgres. Ползва се само вътрешно от playbook-а
# (вие никога не я пишете на ръка някъде). Генерирайте я и я забравете.
# Генериране: pwgen -s 64 1
postgres_connection_password: ''

########################################################################
# authentik
########################################################################

# Включва самия authentik (server + worker контейнери).
authentik_enabled: true

# Домейнът, на който ще отваряте authentik в браузъра. Трябва да имате
# DNS запис (или ред в /etc/hosts на лаптопа ви), който сочи към IP-то на VM-а.
authentik_hostname: sso-test.example.com

# Таен ключ, с който authentik подписва бисквитките на сесиите.
# Генерира се веднъж; смяната му по-късно разлогва всички потребители.
# Генериране: pwgen -s 64 1
authentik_secret_key: ''

# --- Първоначален администратор (bootstrap) ---
# Тези три реда създават админ акаунта "akadmin" автоматично при ПЪРВОТО
# стартиране, за да не минавате ръчно през setup екрана в браузъра.
# Действат само на чиста инсталация — после промяната им няма ефект.

# Имейлът на админ акаунта (с него се логвате).
authentik_bootstrap_email: admin@example.com

# Паролата на админ акаунта. Това е тестова среда — сложете нещо просто,
# но НЕ преизползвайте истинска ваша парола.
authentik_bootstrap_password: ''

# По желание: готов API token за админа. Удобен е за проверките по-долу
# (curl към API-то без логин през браузър). Генериране: pwgen -s 48 1
authentik_bootstrap_token: ''

# --- Метрики (за тест 3) ---
# Публикува Prometheus метриките на authentik през Traefik на
# https://<hostname>/metrics. В реална среда бихте добавили и Basic Auth
# (authentik_metrics_container_labels_traefik_basicauth_*), за теста не е нужно.
authentik_metrics_enabled: true

# --- Brands (за тест 6) — разкоментирайте СЛЕД като базовата инсталация работи ---
# Brands = различен облик (заглавие, лого) според домейна, от който влизате.
# Първият ред казва на Traefik да приема и втория домейн (иначе заявките
# към него изобщо не стигат до authentik). За него също трябва DNS запис!
# authentik_container_labels_traefik_additional_hostnames:
#   - sso-test2.example.com
#
# Списъкът с брандове: domain = кой домейн какъв облик получава;
# branding_title = заглавието в интерфейса; default: true = резервният бранд,
# който се ползва, когато никой domain не съвпадне (може само един такъв).
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
