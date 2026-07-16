<!--
SPDX-FileCopyrightText: 2026 MASH project contributors

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# План за подобряване на ansible-role-authentik

Дата: 2026-07-15

## Контекст и констатации

Това репо е fork на MASH ролята за authentik, замръзнал на **authentik 2025.4.1**.
Междувременно:

- Официалната MASH роля ([mother-of-all-self-hosting/ansible-role-authentik](https://github.com/mother-of-all-self-hosting/ansible-role-authentik))
  е на **v2026.5.2-0** и точно тази версия е закачена в `templates/requirements.yml`
  на mash-playbook.
- authentik премина през няколко несъвместими промени:
  - **2025.8** — задачите се местят от Celery/Redis в PostgreSQL (Dramatiq);
    `AUTHENTIK_BROKER__*` и `AUTHENTIK_RESULT_BACKEND__*` са премахнати;
    `AUTHENTIK_WORKER__CONCURRENCY` → `AUTHENTIK_WORKER__THREADS`;
    метриките се сервират от **worker** контейнера, не от server.
  - **2025.10** — **Redis е премахнат напълно** (кеш, WebSocket и embedded
    outpost минават към Postgres; ~50% повече връзки към базата).
  - **2025.12+** — storage rework: `/media` → `/data` (`AUTHENTIK_STORAGE__*`).
  - **2026.5** — Rust worker entrypoint (−200 MB памет), listen по подразбиране
    `[::]` вместо `0.0.0.0`, нов `AUTHENTIK_BOOTSTRAP_PASSWORD_HASH`.
- Ъпгрейдът трябва да мине **последователно през major версиите**
  (2025.4 → 2025.6 → 2025.8 → 2025.10 → 2026.2 → 2026.5), като 2025.8 изисква
  внимание при натоварени инстанции (миграция на опашката от задачи).
  Версиите на outpost-ите трябва да се сменят едновременно с инстанцията.

---

## Етап 0 — Синхронизация с upstream (най-голямата печалба)

Препоръка: вместо да пренаписваме същото, **ребазираме fork-а върху
upstream v2026.5.2-0** и върху него наслагваме локалните подобрения.
Локалните комити (renovate.json, autotag.yml, дребни фиксове) вече съществуват
в еквивалентен вид upstream.

Какво носи синхронизацията (разлики спрямо текущото състояние):

1. **authentik 2025.4.1 → 2026.5.2** (+ бележка в CHANGELOG за последователния
   ъпгрейд път и за миграцията на задачите в 2025.8).
2. **Премахване на Redis**: `authentik_config_redis_*`, `CELERY_BROKER_URL` и
   `AUTHENTIK_REDIS__*` от env шаблона; deprecation проверки във
   `validate_config.yml` за всички стари имена.
3. **Worker като самостоятелен контейнер** (`… server:TAG worker`) със свой
   `docker create` — вместо крехкия `docker exec` в server контейнера.
   Собствен healthcheck, мрежи, hardening (`--cap-drop=ALL`, `--read-only`,
   tmpfs), cleanup при старт, `Requires`/`Wants` списъци.
4. **`/media` → `/data`** + миграционна задача, която мести старата директория.
5. **`AUTHENTIK_POSTGRESQL__SSLMODE`** реално се подава (сега
   `authentik_database_sslmode` е дефинирана, но не влиза в env — бъг).
6. **Path prefix поддръжка**: `authentik_path_prefix` + Traefik
   `PathPrefix`/strip-prefix/slashless-redirect middlewares.
7. Модерни MASH конвенции:
   - `authentik_container_image_registry_prefix_upstream(_default)` верига
     (за огледални регистри);
   - `authentik_container_extra_arguments(_auto/_custom)`;
   - `authentik_container_labels_additional_labels_auto/_custom` (списък вместо
     низ);
   - `authentik_restart_necessary` (изчислен от промени по конфигурация/образ) —
     изискван от wiring-а на mash-playbook;
   - image pull през `community.docker.docker_image_pull` или `command` метод;
   - мрежа през `community.docker.docker_network` (не `community.general`);
   - `authentik_container_tmpfs_tmp_size`, stop-grace/cleanup таймаути;
   - `TZ`, `AUTHENTIK_ERROR_REPORTING__ENABLED` променливи;
   - `X-Frame-Options` премахнат (дублира CSP `frame-ancestors`),
     `Permission-Policy` → `Permissions-Policy` (правописен бъг);
   - typo фикс `authentik_custon_templates_path` → `authentik_custom_templates_path`.
8. **Инструментариум**: REUSE/SPDX хедъри, `.pre-commit-config.yaml`,
   `.yamllint.yml`, `.ansible-lint`, `justfile`, `mise.toml`, разширен README,
   поправен `meta/main.yml` (текущото описание е копирано от музикална услуга).

Стъпки:

- `git remote add upstream …/mother-of-all-self-hosting/ansible-role-authentik`
- нов бранч от `upstream/main`; пренасяне на евентуални локални специфики;
- тагване по MASH конвенция `v2026.5.2-0` (autotag вече е наличен).

---

## Етап 1 — Поправка на бъгове, които ги има и в upstream (поправят се в това репо)

1. **`authentik_loglevel` е мъртва променлива** — дефинирана в defaults, но
   `AUTHENTIK_LOG_LEVEL` никога не се записва в env. Да се добави в `env.j2`.
2. **`authentik_database_port` не се подава** — `AUTHENTIK_POSTGRESQL__PORT`
   липсва в env шаблона (работи само защото 5432 е default).
3. **Метриките са на грешния контейнер** — от 2025.8 порт 9300 се сервира от
   worker-а, а Traefik labels за метрики стоят върху server контейнера.
   Решение: отделен labels файл за worker или документиране + преместване.
4. **`AUTHENTIK_EMAIL__TIMEOUT` е хардкоднат на 10** — да стане променлива
   `authentik_email_timeout`.
5. **IPv4-only среди**: от 2026.5 default listen е `[::]` — да се добави
   променлива за `AUTHENTIK_LISTEN__HTTP` и бележка в README.

---

## Етап 2 — Пълно покритие на конфигурацията на authentik

Първокласни променливи (с разумни defaults, всичко останало остава възможно през
`authentik_environment_variables_additional_variables`):

| Група | Променливи |
| --- | --- |
| PostgreSQL | `PORT`, `CONN_MAX_AGE`, `CONN_HEALTH_CHECKS`, `DISABLE_SERVER_SIDE_CURSORS`, `DEFAULT_SCHEMA`, read replicas |
| Web | `WEB__WORKERS`, `WEB__THREADS`, `WEB__PATH` (**задължителен при `authentik_path_prefix != '/'`** — да се подава автоматично!) |
| Worker | `WORKER__PROCESSES`, `WORKER__THREADS` |
| Listen | `LISTEN__TRUSTED_PROXY_CIDRS` (важно зад Traefik за верни клиентски IP-та) |
| Storage | `STORAGE__BACKEND` file/s3 + пълен S3 набор (endpoint, region, bucket, ключове, custom domain) — media върху S3 |
| Bootstrap | `AUTHENTIK_BOOTSTRAP_EMAIL/PASSWORD/TOKEN` + `PASSWORD_HASH` (2026.5) — автоматизирано първоначално админ-сетване без ръчен initial-setup flow |
| Сесии/кеш | `SESSIONS__UNAUTHENTICATED_AGE`, `CACHE__TIMEOUT*`, `REPUTATION__EXPIRY` |
| Разни | `COOKIE_DOMAIN`, `DISABLE_UPDATE_CHECK`, `EMAIL__TIMEOUT`, `LOG_LEVEL` |
| GeoIP | `EVENTS__CONTEXT_PROCESSORS__GEOIP/ASN` + опционален bind mount на MaxMind бази от хоста |
| Outposts | `OUTPOSTS__CONTAINER_IMAGE_BASE`, `OUTPOSTS__DISCOVER` |

---

## Етап 3 — Нови възможности отвъд upstream

1. **Outpost под-услуги** (най-голяма добавена стойност; MASH docs изрично
   казват, че LDAP не е тестван):
   - LDAP outpost (`ghcr.io/goauthentik/ldap`) — отделен systemd service,
     `AUTHENTIK_HOST/TOKEN/INSECURE`, портове 3389/6636;
   - RADIUS outpost (`ghcr.io/goauthentik/radius`);
   - Proxy outpost (`ghcr.io/goauthentik/proxy`) за forward auth на услуги извън
     embedded outpost-а;
   - обща бележка: версията на outpost = версията на инстанцията (обвързване с
     `authentik_version`).
2. **Blueprints** — mount на `/blueprints` + възможност playbook-ът да подава
   декларативни YAML blueprint файлове (flows, приложения, provider-и) →
   възпроизводима конфигурация без кликане в UI.
3. **Мониторинг wiring** — интеграция с `mash_playbook_metrics_exposure_*`
   шаблона (Prometheus scrape на worker:9300, опционална basic auth защита).
4. **`AUTHENTIK_LISTEN__METRICS` / debug портове** — изключване/ограничаване.

---

## Етап 4 — Документация (само в това репо)

1. README на ролята по MASH стандарт + CHANGELOG записи за всички breaking
   changes (Redis, /data, ъпгрейд път).
2. Примери в README за употреба с MASH playbook: forward auth през Traefik,
   LDAP/RADIUS outposts, bootstrap променливи. Всички промени остават в това
   репо — не се предвиждат PR-и към mash-playbook или upstream ролята.

---

## Етап 5 — Качество

- ansible-lint + yamllint + pre-commit (идват с Етап 0), Woodpecker CI;
- REUSE compliance (SPDX хедъри навсякъде);
- опционално: molecule сценарий за smoke test (server + worker стартират,
  healthcheck зелен);
- Renovate поддържа `authentik_version` актуален (вече конфигуриран).

## Препоръчан ред на изпълнение

1. Етап 0 (синхронизация) — незабавно; всичко друго стъпва на него.
2. Етап 1 (бъгове) — дребни и бързи поправки.
3. Етап 2 + Етап 3.1 (outposts) — основната нова функционалност.
4. Етапи 3.2–5 — по приоритет.
