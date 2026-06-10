# Production deployment strategy

## Цель

Документ описывает production-level flow для Aniyume после разделения проекта на отдельные репозитории:

- **Aniyume Web** — пользовательский Next.js frontend;
- **Aniyume API** — Laravel API и backend runtime;
- **Aniyume Admin Web** — отдельный admin frontend.

Цель стратегии — выкатывать `release` ветки отдельных репозиториев на production Docker host без лишнего усложнения, сохранив понятный rollback и разделение ответственности между application stacks и общим infrastructure stack.

Документ не требует изменений runtime-файлов, compose-файлов или кода приложения. Это архитектурный план для production deployment.

## Branch and environment model

### Branches

| Branch | Назначение | Где используется |
|---|---|---|
| `dev` | интеграционная ветка разработки | local/dev/staging-like окружения |
| `release` | production-ready ветка | production Docker host |

Правило: production деплоит только артефакты, собранные из `release`. Прямой деплой из `dev` на production запрещён.

### Environments

| Environment | Source branch | Назначение | Характеристики |
|---|---|---|---|
| Local/dev | `dev` | разработка и проверка фич | bind mounts, dev server, debug env, локальные порты |
| Staging/pre-release, опционально | `release` или release candidate | smoke перед production | production-like images, отдельные env/secrets, тестовые домены |
| Production | `release` | публичный сервис | immutable Docker images, `APP_ENV=production`, `APP_DEBUG=false`, TLS, backups, health checks |

Минимальная схема может стартовать без отдельного staging, но production flow должен быть построен так, чтобы staging можно было добавить позже без переработки архитектуры.

## High-level production architecture

```text
Internet
  |
  | HTTPS / WSS
  v
Reverse proxy / edge nginx / Traefik / Caddy
  |---------------------------> aniyume-web:3000
  |---------------------------> aniyume-admin-web:3001
  |---------------------------> aniyume-api:8000 or api-nginx/php-fpm
  |---------------------------> aniyume-reverb:8080
                                  |
                                  v
                         aniyume-db:5432
                         aniyume-redis:6379

Background backend containers use the same API image:
  aniyume-queue-worker  -> db + redis
  aniyume-scheduler     -> db + redis
```

### Service inventory

| Service | Ownership | Production responsibility | Exposed publicly |
|---|---|---|---|
| `reverse-proxy` | infra stack | TLS termination, HTTP routing, WebSocket upgrade, security headers | yes, ports `80/443` |
| `web` | Aniyume Web repo | Next.js user-facing application | through proxy only |
| `admin-web` | Aniyume Admin Web repo | admin UI | through proxy only |
| `api` | Aniyume API repo | Laravel HTTP API runtime | through proxy only |
| `queue-worker` | Aniyume API repo | Laravel queued jobs | no |
| `scheduler` | Aniyume API repo | Laravel scheduled tasks | no |
| `reverb` | Aniyume API repo | Laravel Reverb WebSocket server | through proxy only |
| `db` | infra stack | PostgreSQL persistent data | no |
| `redis` | infra stack | cache/session/queue broker | no |

## What deploys separately vs common infra

### Separate application deployments

Each application repository should build and publish its own versioned Docker image from `release`:

| Repository | Images | Typical containers on host |
|---|---|---|
| Aniyume Web | `registry.example.com/aniyume/web:<tag>` | `web` |
| Aniyume API | `registry.example.com/aniyume/api:<tag>` | `api`, `queue-worker`, `scheduler`, `reverb`, one-off migration job |
| Aniyume Admin Web | `registry.example.com/aniyume/admin-web:<tag>` | `admin-web` |

Recommended tag format:

```text
<repo>-release-<yyyyMMddHHmm>-<short_sha>
```

Also maintain moving tags if convenient:

```text
release
release-previous
```

For rollback, immutable SHA/version tags are more important than moving tags.

### Common infra stack

The production host should keep common stateful and edge services in a separate infra stack:

| Infra component | Reason to keep outside app repos |
|---|---|
| Reverse proxy | shared routing, TLS, cert renewal, rate limits, common headers |
| PostgreSQL | persistent state, backups, lifecycle independent of app release |
| Redis | shared cache/queue broker, lifecycle independent of app image |
| Docker networks | stable service discovery between stacks |
| Volumes/backups | should not be recreated during app rollback |
| Monitoring/log shipping, later | shared operational concern |

This allows Web/Admin/API to be redeployed independently without recreating database volumes or TLS/proxy configuration.

## Recommended Docker stack split

### 1. `aniyume-infra` stack

Long-lived stack, changed rarely:

```text
reverse-proxy
db
redis
external Docker network: aniyume-public
internal Docker network: aniyume-internal
volumes: db-data, redis-data, api-storage
```

Notes:

- `db` and `redis` must not expose public ports on production.
- `db-data` must be included in backup policy.
- `api-storage` should persist Laravel uploaded/generated public files if the app uses local storage.
- Reverse proxy connects to `aniyume-public`; API/background services connect to `aniyume-internal`.

### 2. `aniyume-api` stack

Deployed from Aniyume API `release` image:

```text
api
queue-worker
scheduler
reverb
one-off migration container during deploy
```

All containers use the same image tag to avoid code drift:

```text
registry.example.com/aniyume/api:<release_tag>
```

Runtime commands differ:

- `api`: Laravel HTTP runtime;
- `queue-worker`: `php artisan queue:work --sleep=3 --tries=3 --timeout=300`;
- `scheduler`: `php artisan schedule:work` or a controlled `schedule:run` loop;
- `reverb`: `php artisan reverb:start --host=0.0.0.0 --port=8080`;
- migration job: `php artisan migrate --force`.

### 3. `aniyume-web` stack

Deployed from Aniyume Web `release` image:

```text
web
```

The container runs Next.js production server on an internal port, usually `3000`, and is reachable only from reverse proxy.

### 4. `aniyume-admin-web` stack

Deployed from Aniyume Admin Web `release` image:

```text
admin-web
```

The container runs the admin frontend on an internal port, usually `3001`, and is reachable only from reverse proxy.

## Domains and routing

### Recommended production domains

For clarity and minimum routing ambiguity, use separate subdomains:

| Purpose | Domain | Routes to |
|---|---|---|
| User web | `aniyume.example.com` or apex `example.com` | `web:3000` |
| API | `api.aniyume.example.com` or `api.example.com` | `api:8000` |
| Admin web | `admin.aniyume.example.com` or `admin.example.com` | `admin-web:3001` |
| WebSocket/Reverb | `ws.aniyume.example.com` or `ws.example.com` | `reverb:8080` |

Best practical option for production:

```text
https://aniyume.example.com        -> Web
https://api.aniyume.example.com    -> API
https://admin.aniyume.example.com  -> Admin Web
wss://ws.aniyume.example.com       -> Reverb
```

The project uses the apex domain `aniyume.tech` for Web:

```text
https://aniyume.tech
https://api.aniyume.tech
https://admin.aniyume.tech
wss://ws.aniyume.tech
```

### Why separate domains are preferred

- avoids fragile path rewrites between Next.js `/api/external/*` and Laravel `/api/v1`;
- makes CORS and cookie/security policies explicit;
- lets admin have stricter access control later;
- lets WebSocket routing be configured independently with upgrade headers;
- simplifies rollback of Web/Admin/API independently.

### Alternative path-based layout

Path-based layout is possible but less preferred:

```text
https://aniyume.example.com/        -> Web
https://aniyume.example.com/api/*   -> API
https://aniyume.example.com/admin/* -> Admin Web
https://aniyume.example.com/app/*   -> Reverb
```

Use it only if DNS/subdomains are constrained. It requires strict alignment with existing frontend proxy path `/api/external/*` and Laravel API prefix `/api/v1`.

## Reverse proxy strategy

Use one production reverse proxy on the Docker host. Good choices:

- **Nginx** — explicit and predictable, good default;
- **Traefik** — convenient if many stacks and labels are preferred;
- **Caddy** — simple automatic TLS, good for small deployments.

Recommended initial choice: **Nginx or Caddy**. Avoid Kubernetes or service mesh at this stage.

### Reverse proxy responsibilities

- terminate TLS for all public domains;
- redirect HTTP to HTTPS;
- route hostnames to internal Docker services;
- pass `X-Forwarded-*` headers;
- support WebSocket upgrade for Reverb;
- set upload/body/timeouts appropriate for API and media flows;
- optionally apply rate limits to auth/API endpoints;
- keep `/nginx-health` or equivalent local health endpoint.

### Nginx-style routing sketch

```nginx
# Web
server {
    listen 443 ssl http2;
    server_name aniyume.example.com;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://aniyume-web:3000;
    }
}

# API
server {
    listen 443 ssl http2;
    server_name api.aniyume.example.com;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://aniyume-api:8000;
    }
}

# Admin Web
server {
    listen 443 ssl http2;
    server_name admin.aniyume.example.com;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_pass http://aniyume-admin-web:3001;
    }
}

# Reverb WebSocket
server {
    listen 443 ssl http2;
    server_name ws.aniyume.example.com;

    location /app/ {
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        proxy_pass http://aniyume-reverb:8080;
    }
}
```

The actual Docker service names may differ; the important rule is that proxy uses stable network aliases, not container IDs.

## Required environment variables

Secrets must be stored in CI/CD variables, Docker secrets, host-level protected env files, or a secret manager. Do not commit real production `.env` values.

### API / Laravel env

```env
APP_NAME=Aniyume
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://api.aniyume.example.com

LOG_CHANNEL=stderr
LOG_LEVEL=info

DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=aniyume
DB_USERNAME=aniyume
DB_PASSWORD=...

CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=...

BROADCAST_CONNECTION=reverb
REVERB_APP_ID=...
REVERB_APP_KEY=...
REVERB_APP_SECRET=...
REVERB_HOST=0.0.0.0
REVERB_PORT=8080
REVERB_SCHEME=http

# Browser-facing values if backend also renders/broadcasts URLs
REVERB_PUBLIC_HOST=ws.aniyume.example.com
REVERB_PUBLIC_PORT=443
REVERB_PUBLIC_SCHEME=https

FRONTEND_URL=https://aniyume.example.com
ADMIN_URL=https://admin.aniyume.example.com
SANCTUM_STATEFUL_DOMAINS=aniyume.example.com,admin.aniyume.example.com
SESSION_DOMAIN=.aniyume.example.com
CORS_ALLOWED_ORIGINS=https://aniyume.example.com,https://admin.aniyume.example.com

# External integrations, if used by the application
MAIL_MAILER=...
MAIL_HOST=...
MAIL_USERNAME=...
MAIL_PASSWORD=...
PAYMENT_PROVIDER_KEY=...
ANIME_EXTERNAL_API_KEYS=...
```

If the current production choice remains database queues, use `QUEUE_CONNECTION=database`; however Redis is preferable for production Docker because it separates queue workload from PostgreSQL OLTP traffic.

### Web / Next.js env

```env
NODE_ENV=production
BACKEND_URL=https://api.aniyume.example.com/api/v1

NEXT_PUBLIC_API_BASE=/api/external
NEXT_PUBLIC_REVERB_APP_KEY=...
NEXT_PUBLIC_REVERB_HOST=ws.aniyume.example.com
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

`BACKEND_URL` is server-side/proxy-facing. Public values prefixed with `NEXT_PUBLIC_` are embedded into browser bundles, so they must not contain secrets.

### Admin Web env

```env
NODE_ENV=production
ADMIN_BACKEND_URL=https://api.aniyume.example.com/api/v1
NEXT_PUBLIC_ADMIN_API_BASE=https://api.aniyume.example.com/api/v1

NEXT_PUBLIC_REVERB_APP_KEY=...
NEXT_PUBLIC_REVERB_HOST=ws.aniyume.example.com
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

Admin must use the same auth model as API supports. If Bearer tokens remain canonical, avoid introducing cookie-only admin auth in deployment configuration.

### Infra env

```env
POSTGRES_DB=aniyume
POSTGRES_USER=aniyume
POSTGRES_PASSWORD=...

REDIS_PASSWORD=...

LETSENCRYPT_EMAIL=ops@example.com
WEB_DOMAIN=aniyume.example.com
API_DOMAIN=api.aniyume.example.com
ADMIN_DOMAIN=admin.aniyume.example.com
WS_DOMAIN=ws.aniyume.example.com
```

## Production deploy flow

### 1. Prepare release branch

For each repository:

1. merge tested changes from `dev` to `release`;
2. run repository CI quality gates;
3. require green build/test status before image publishing;
4. tag the image with immutable release tag based on commit SHA.

Suggested minimum gates:

| Repo | Gates |
|---|---|
| Aniyume Web | install, lint/typecheck if available, `npm run build` |
| Aniyume Admin Web | install, lint/typecheck if available, `npm run build` |
| Aniyume API | composer install, PHP tests, route/schedule validation, migration preview if possible |

### 2. Build and publish images

CI builds production images from `release` and pushes them to registry:

```text
registry.example.com/aniyume/web:<web_release_tag>
registry.example.com/aniyume/api:<api_release_tag>
registry.example.com/aniyume/admin-web:<admin_release_tag>
```

The production host should pull only these images. It should not build from mutable working directories.

### 3. Deploy on Docker host

Recommended deployment unit: a versioned deploy manifest on the production host or in an infra repo, for example:

```env
WEB_IMAGE=registry.example.com/aniyume/web:web-release-202605261200-a1b2c3d
API_IMAGE=registry.example.com/aniyume/api:api-release-202605261205-d4e5f6a
ADMIN_IMAGE=registry.example.com/aniyume/admin-web:admin-release-202605261210-1a2b3c4
```

Deployment sequence:

1. confirm database backup exists before API deployments with migrations;
2. pull new images;
3. start or update API application containers except workers if migration compatibility requires a pause;
4. run migrations as an explicit one-off API container command;
5. restart/update `api`, `queue-worker`, `scheduler`, `reverb` on the same API image tag;
6. update `web` and/or `admin-web` images;
7. keep reverse proxy and infra volumes unchanged;
8. run health checks and smoke checks.

### 4. Migrations

Migrations are part of the API release, not Web/Admin releases.

Rules:

- never run migrations implicitly on every container boot;
- run `php artisan migrate --pretend` in CI or pre-release where possible;
- create a database backup before production migrations;
- run production migration explicitly with `php artisan migrate --force`;
- prefer backward-compatible migrations: expand first, deploy app, contract later.

Safe migration deployment pattern:

```text
1. API release N adds nullable/new columns or new tables.
2. Deploy API N and run migrations.
3. Deploy Web/Admin that can use the new API behavior.
4. In a later release, remove old columns/paths only after all apps stop using them.
```

### 5. Health checks

Minimum post-deploy checks:

| Component | Check |
|---|---|
| Reverse proxy | HTTPS responds, certificates valid, HTTP redirects to HTTPS |
| Web | `GET https://aniyume.example.com/` returns 200/valid HTML |
| API | `GET https://api.aniyume.example.com/up` and a lightweight public API endpoint |
| Admin | `GET https://admin.aniyume.example.com/` returns 200/valid HTML |
| Reverb | WebSocket handshake through `wss://ws.aniyume.example.com/app/...` |
| Queue worker | process running, no repeated restart loop, can process a test/known job |
| Scheduler | process running, `schedule:list` valid in same image/env |
| DB | migrations table updated, app can read/write expected data |
| Redis | `PING` through internal network, queue/cache reachable |

Functional smoke after important releases:

- login/register;
- catalog load;
- anime detail/player load;
- profile/watch history/favorites;
- comments/ratings;
- Watch Party create/join/message sync;
- admin login and basic list page;
- background import job visibility if affected.

## Independent deployment rules

### Web-only release

Use when UI changes do not require API changes.

Flow:

1. merge Web `dev` -> `release`;
2. build/publish Web image;
3. update only `web` container;
4. smoke `https://aniyume.example.com` and API calls through frontend.

No DB migrations.

### Admin-only release

Use when admin UI changes do not require API changes.

Flow:

1. merge Admin `dev` -> `release`;
2. build/publish Admin image;
3. update only `admin-web` container;
4. smoke admin auth and key admin pages.

No DB migrations.

### API release

Use when backend API, jobs, scheduler, Reverb, or DB schema changes.

Flow:

1. merge API `dev` -> `release`;
2. build/publish API image;
3. create/verify DB backup if migrations exist;
4. run migration one-off;
5. update `api`, `queue-worker`, `scheduler`, `reverb` to the same image tag;
6. smoke API, workers, scheduler, Reverb;
7. verify Web/Admin compatibility.

### Coordinated release

Use when API contract changes require Web/Admin updates.

Preferred order:

1. deploy backward-compatible API first;
2. run migrations;
3. deploy Web/Admin;
4. verify all smoke checks;
5. only in a later release remove deprecated API behavior.

Avoid simultaneous breaking changes unless maintenance window and rollback plan are explicit.

## Rollback strategy

### Application rollback

Rollback should switch containers back to previously known-good image tags.

Keep a small release ledger on the host or in infra repo:

```text
current:
  web: web-release-202605261200-a1b2c3d
  api: api-release-202605261205-d4e5f6a
  admin-web: admin-release-202605261210-1a2b3c4
previous:
  web: web-release-202605241830-abc1111
  api: api-release-202605241845-def2222
  admin-web: admin-release-202605241900-ghi3333
```

Rollback by component:

| Failed component | Rollback action |
|---|---|
| Web | redeploy previous Web image only |
| Admin Web | redeploy previous Admin image only |
| API without migration | redeploy previous API image to `api`, `queue-worker`, `scheduler`, `reverb` |
| Reverb only | redeploy previous API image or restart `reverb` if image is correct |
| Worker issue | stop workers, redeploy previous API image for workers, then API if needed |

After rollback:

- restart affected containers;
- clear Laravel caches if env/config changed;
- run health checks;
- verify no queue job incompatibility remains.

### Database rollback

Database rollback is the risky part and must be treated separately.

Preferred policy:

- design migrations to be backward-compatible;
- do not rely on `migrate:rollback` for destructive production changes;
- take backup before migrations;
- for data-destructive migrations, rollback means restoring backup or applying a reviewed forward-fix migration.

Rollback decision matrix:

| Situation | Recommended action |
|---|---|
| Migration added table/nullable column and old app ignores it | rollback app image only, keep schema |
| Migration changed data format but old app can still read it | rollback app image, monitor closely |
| Migration dropped/renamed columns used by old app | do not blindly rollback app; restore DB backup or deploy forward fix |
| Bad seed/config data | apply corrective migration/SQL after backup |

### Rollback time target

For non-DB app failures, target rollback should be simple and fast:

```text
update image tag -> recreate affected service -> health check
```

No infra stack recreation should be required.

## Backups and persistence

Minimum production persistence policy:

- PostgreSQL daily automated backups plus pre-migration backup;
- backup restore procedure tested periodically;
- Laravel storage volume backed up if it contains user uploads or generated assets;
- Redis persistence optional for cache-only, recommended if queues must survive restarts;
- never remove Docker volumes during application deploy or rollback.

## Security baseline

- `APP_DEBUG=false` in production.
- No public ports for DB, Redis, API internals, workers, scheduler, or Reverb direct port.
- TLS for all public domains.
- Secrets are not stored in git or Docker images.
- Admin domain should be isolated and can later get IP allowlist, VPN, or basic edge auth.
- CORS should allow only Web/Admin production origins.
- Reverse proxy should pass correct `X-Forwarded-Proto` so Laravel generates HTTPS URLs.

## Minimal hosting path for release branches

Recommended simple path to production Docker host:

1. Create production infra stack once: reverse proxy, DB, Redis, networks, volumes, TLS.
2. Configure DNS for Web/API/Admin/WS domains to the host.
3. Configure production env/secrets on host or CI secret store.
4. For each repo, CI builds Docker image on push/merge to `release`.
5. CI pushes immutable image tag to registry.
6. Deployment updates only the affected application stack on the host.
7. API deploy runs migrations explicitly and updates API-related containers together.
8. Health checks decide success or rollback.
9. Rollback uses previous image tags and does not recreate infra/stateful services.

This gives production-level separation without introducing Kubernetes, blue/green orchestration, or complex release tooling before the project needs it.

## Operational checklist

Before first production release:

- [ ] production DNS records created;
- [ ] TLS certificates issued/auto-renewed;
- [ ] Docker registry access configured on host;
- [ ] infra stack running: reverse proxy, PostgreSQL, Redis;
- [ ] production secrets configured outside git;
- [ ] DB backup job configured and restore path documented;
- [ ] Web/API/Admin images built from `release`;
- [ ] API migrations tested in production-like env;
- [ ] reverse proxy routes Web/API/Admin/WS correctly;
- [ ] health checks and smoke checklist agreed by team;
- [ ] previous image tags retained for rollback.
