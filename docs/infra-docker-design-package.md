# Infra Docker design package

## Цель и границы документа

Документ фиксирует целевую Docker/Compose-архитектуру Aniyume для следующей фазы реализации. На этом этапе **Dockerfile, compose-файлы и runtime-конфигурация не создаются**: пакет нужен как согласованный дизайн, чтобы последующая реализация не конфликтовала с параллельными изменениями в backend/frontend/admin.

Область проектирования:

- `backend` — Laravel API;
- `frontend` — Next.js пользовательский интерфейс;
- `admin` — future admin UI или отдельный admin runtime;
- `db` — PostgreSQL;
- `redis` — cache/session/queue backend для production-like сценария;
- `queue-worker` — Laravel queue worker;
- `scheduler` — Laravel scheduler;
- `reverb` — Laravel Reverb websocket server;
- `nginx` / reverse proxy — единая точка входа.

Важные ограничения:

- публичные API endpoints не переименовываются;
- frontend-facing path `/api/external/*` сохраняется;
- backend API prefix `/api/v1` сохраняется;
- production layout должен поддерживать WebSocket upgrade для Reverb;
- реальные `.env` с секретами не должны попадать в git.

## Целевая схема сервисов

```text
Browser
  |
  | HTTP / HTTPS / WebSocket
  v
nginx / reverse proxy
  |-------------------------------> frontend:3000
  |-------------------------------> future admin:3001
  |-------------------------------> backend php runtime:8000 or php-fpm:9000
  |-------------------------------> reverb:8080
                                  |
backend / worker / scheduler / reverb
  |-------------------------------> db:5432
  |-------------------------------> redis:6379
```

Recommended external routing:

```text
http://localhost              -> frontend
http://localhost/admin        -> future admin, when implemented
http://localhost/api/*        -> backend API proxy to /api/v1 or direct backend path rules
ws://localhost/app/*          -> Reverb websocket upgrade
```

Production-like domain layout may keep the existing deployment recommendation:

```text
https://example.com           -> frontend
https://admin.example.com     -> future admin
https://api.example.com       -> Laravel backend
wss://ws.example.com          -> Laravel Reverb
```

## Service inventory

### `backend`

Purpose: Laravel HTTP API runtime.

Implementation options for the next phase:

1. **Dev-friendly single container**: PHP CLI server or Laravel `artisan serve`.
2. **Production-like split**: PHP-FPM container behind nginx, or Laravel Octane only if the application is explicitly prepared for it later.

Recommended initial implementation: PHP 8.2+ image with Composer, application source mounted in dev and copied in production-like image.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `8000` | `8000` optional | Direct debug access; nginx should still be primary entrypoint. |
| Production-like | `9000` or `8000` | none | Exposed only to `nginx` network. |

Startup command candidates:

```bash
# dev option
php artisan serve --host=0.0.0.0 --port=8000

# production-like php-fpm option
php-fpm
```

Key env variables:

```env
APP_NAME=Aniyume
APP_ENV=local|production
APP_KEY=base64:...
APP_DEBUG=true|false
APP_URL=http://localhost|https://api.example.com

DB_CONNECTION=pgsql
DB_HOST=db
DB_PORT=5432
DB_DATABASE=aniyume
DB_USERNAME=aniyume
DB_PASSWORD=change_me

CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=null

BROADCAST_CONNECTION=reverb
REVERB_APP_ID=aniyume
REVERB_APP_KEY=change_me
REVERB_APP_SECRET=change_me
REVERB_HOST=reverb
REVERB_PORT=8080
REVERB_SCHEME=http
```

Volumes:

- dev: bind mount backend source into container;
- dev/prod-like persistent writable paths:
  - `aniyume-backend/storage`;
  - `aniyume-backend/bootstrap/cache`;
- optional Composer cache volume for faster local builds.

Dependencies:

- `db` must be accepting connections before migrations/API traffic;
- `redis` must be healthy when Redis-backed cache/session/queue is enabled;
- `backend` should not require `queue-worker`/`scheduler` to boot.

Healthcheck candidates:

```bash
# HTTP runtime
curl -fsS http://localhost:8000/up || curl -fsS http://localhost:8000/api/v1/public/anime

# php-fpm runtime
php -v
```

If Laravel has no stable lightweight health endpoint, add one in a later application phase instead of using a heavy catalog endpoint.

### `frontend`

Purpose: Next.js user-facing application.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `3000` | `3000` optional | Direct Next.js access for HMR/debug. |
| Production-like | `3000` | none | Exposed through nginx only. |

Startup command candidates:

```bash
# dev
npm run dev -- --hostname 0.0.0.0 --port 3000

# production-like
npm run build
npm run start -- --hostname 0.0.0.0 --port 3000
```

Key env variables:

```env
BACKEND_URL=http://backend:8000/api/v1

NEXT_PUBLIC_REVERB_APP_KEY=change_me
NEXT_PUBLIC_REVERB_HOST=localhost
NEXT_PUBLIC_REVERB_PORT=80
NEXT_PUBLIC_REVERB_SCHEME=http
```

For production-like deployment behind TLS:

```env
BACKEND_URL=https://api.example.com/api/v1
NEXT_PUBLIC_REVERB_HOST=ws.example.com
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

Volumes:

- dev: bind mount frontend source;
- dev: named volume for `node_modules` to avoid host/container platform conflicts;
- optional Next.js cache volume for faster rebuilds.

Dependencies:

- `backend` for server-side API proxy calls;
- `reverb` only for realtime browser flows, not for process boot.

Healthcheck candidate:

```bash
curl -fsS http://localhost:3000/ || exit 1
```

### `admin` / future admin

Purpose: reserved service slot for future admin UI or separated admin runtime.

Current design decision: include the service in architecture and env/routing plan, but keep actual implementation optional until admin application ownership is clear.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `3001` | `3001` optional | If future admin is a separate Next/Vite app. |
| Production-like | `3001` | none | Exposed via nginx, e.g. `/admin` or `admin.example.com`. |

Startup command candidates:

```bash
# placeholder for future separate admin frontend
npm run dev -- --host 0.0.0.0 --port 3001
npm run start -- --host 0.0.0.0 --port 3001
```

Key env variables:

```env
ADMIN_BACKEND_URL=http://backend:8000/api/v1
NEXT_PUBLIC_ADMIN_BACKEND_URL=/api/external
```

Dependencies:

- `backend`;
- auth/session strategy must remain compatible with existing Laravel Sanctum Bearer token flow unless explicitly redesigned later.

Healthcheck candidate:

```bash
curl -fsS http://localhost:3001/ || exit 1
```

Risk: current admin routes are documented as Laravel admin web routes, while future admin may become a separate frontend. The Docker implementation phase must first confirm the owner and runtime of admin UI.

### `db`

Purpose: PostgreSQL database.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `5432` | `5432` or `5433` | External port optional; use `5433` if local PostgreSQL already uses `5432`. |
| Production-like | `5432` | none | Internal network only. |

Image candidate:

```text
postgres:16-alpine
```

Key env variables:

```env
POSTGRES_DB=aniyume
POSTGRES_USER=aniyume
POSTGRES_PASSWORD=change_me
```

Volumes:

- named volume `db-data:/var/lib/postgresql/data`;
- optional init scripts directory for local bootstrap only, if needed later.

Healthcheck:

```bash
pg_isready -U aniyume -d aniyume
```

Dependencies:

- no app-level dependency;
- application services should use Compose `depends_on` with `service_healthy` where supported.

### `redis`

Purpose: cache/session/queue broker for production-like architecture; optional but recommended in dev to match runtime behavior.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `6379` | `6379` optional | Direct debugging with redis-cli if needed. |
| Production-like | `6379` | none | Internal network only. |

Image candidate:

```text
redis:7-alpine
```

Command candidate:

```bash
redis-server --appendonly yes
```

Volumes:

- named volume `redis-data:/data` if persistence is desired;
- no volume if Redis is used as disposable cache only.

Healthcheck:

```bash
redis-cli ping
```

### `queue-worker`

Purpose: Laravel queue processing for imports and background jobs.

Ports: none.

Startup command:

```bash
php artisan queue:work --sleep=3 --tries=3 --timeout=300
```

Key env variables: same as `backend`, with emphasis on:

```env
QUEUE_CONNECTION=redis
DB_HOST=db
REDIS_HOST=redis
```

Volumes:

- same backend source/image as `backend`;
- writable `storage` for logs/cache;
- no public assets volume required unless jobs write user-facing files.

Dependencies:

- `db` healthy;
- `redis` healthy;
- backend image/build available, but the HTTP `backend` container itself does not need to be healthy for the worker to start.

Healthcheck candidates:

```bash
php artisan queue:monitor default --max=100
```

If `queue:monitor` is not configured or too strict, prefer a simple process healthcheck in Compose and application-level monitoring later.

### `scheduler`

Purpose: run Laravel scheduled tasks from `aniyume-backend/routes/console.php`.

Ports: none.

Startup command options:

```bash
# Laravel scheduler loop, preferred for containers if supported by app version
php artisan schedule:work

# alternative shell loop if schedule:work is not acceptable
while true; do php artisan schedule:run --verbose --no-interaction; sleep 60; done
```

Expected scheduled tasks from current documentation:

```text
import:anime
import:episodes --limit=200
episodes:sync-ongoing --days=1
```

Key env variables: same as `backend`.

Volumes:

- same backend source/image as `backend`;
- writable `storage` for logs.

Dependencies:

- `db` healthy;
- `redis` healthy if scheduled tasks dispatch queued jobs or use Redis cache;
- external anime/video APIs available at runtime.

Healthcheck candidates:

```bash
php artisan schedule:list
```

For production-like Compose, container restart policy may be more reliable than a strict scheduler healthcheck.

### `reverb`

Purpose: Laravel Reverb WebSocket server for Watch Party realtime features.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `8080` | `8080` optional | Direct websocket debug access. |
| Production-like | `8080` | none | Exposed through nginx with upgrade headers. |

Startup command:

```bash
php artisan reverb:start --host=0.0.0.0 --port=8080
```

Key env variables:

```env
BROADCAST_CONNECTION=reverb
REVERB_APP_ID=aniyume
REVERB_APP_KEY=change_me
REVERB_APP_SECRET=change_me
REVERB_HOST=0.0.0.0
REVERB_PORT=8080
REVERB_SCHEME=http
```

For browser-facing env behind nginx/TLS:

```env
NEXT_PUBLIC_REVERB_HOST=ws.example.com
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

Volumes:

- same backend source/image as `backend`;
- writable `storage` for logs.

Dependencies:

- `backend` code/image;
- `redis` only if broadcast/cache/session config requires it;
- `db` if auth/private/presence channel authorization path touches database.

Healthcheck candidates:

```bash
php artisan reverb:restart --help
```

Practical note: WebSocket healthchecks are often better handled at nginx/application monitoring level. A process-level healthcheck may be enough for the initial Compose phase.

### `nginx` / reverse proxy

Purpose: single external entrypoint, static/proxy routing, WebSocket upgrade support.

Ports:

| Scenario | Internal | External | Notes |
|---|---:|---:|---|
| Dev | `80` | `80` or `8088` | Use `8088` if local port 80 is occupied. |
| Production-like | `80`, `443` | `80`, `443` | TLS termination in nginx or upstream load balancer. |

Volumes:

- nginx config file(s), read-only;
- TLS certs for production-like local tests if needed;
- optional public/static mount only if Laravel/Next static assets are served directly by nginx.

Dependencies:

- `frontend`;
- `backend`;
- `reverb`;
- future `admin` when enabled.

Healthcheck:

```bash
nginx -t
curl -fsS http://localhost/ || exit 1
```

Required proxy behavior:

```nginx
# frontend
location / {
    proxy_pass http://frontend:3000;
}

# backend API, exact rewrite rules to be confirmed during implementation
location /api/ {
    proxy_pass http://backend:8000;
}

# reverb websocket
location /app/ {
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://reverb:8080;
}
```

Risk: `/api/*` proxy rules must be aligned with the existing Next.js `/api/external/*` proxy and Laravel `/api/v1` prefix. Do not introduce a breaking path rewrite without explicit API contract review.

## Development scenario

Goal: fast local development with mounted source code, optional direct ports, and minimal image rebuilds.

Recommended characteristics:

- Compose file: `docker-compose.yml` + `docker-compose.override.yml` or `compose.dev.yml`;
- source code mounted into `backend` and `frontend` containers;
- `frontend` runs `npm run dev` with HMR;
- `backend` runs `php artisan serve` or php-fpm with nginx;
- `db` and `redis` use named volumes;
- external ports available for debugging:
  - nginx: `80` or `8088`;
  - frontend: `3000` optional;
  - backend: `8000` optional;
  - admin: `3001` optional when implemented;
  - reverb: `8080` optional;
  - db: `5432` or `5433` optional;
  - redis: `6379` optional.

Dev startup order:

1. start `db` and `redis`;
2. install/build app dependencies inside images or volumes;
3. start `backend`;
4. run `php artisan migrate` manually or as an explicit one-off task;
5. start `queue-worker`, `scheduler`, `reverb`;
6. start `frontend` and `nginx`;
7. enable `admin` later when its runtime is finalized.

Do **not** hide migrations inside the long-running backend startup command for dev unless the team explicitly accepts automatic schema changes.

## Production-like scenario

Goal: local/staging deployment close to production without dev HMR and with fewer exposed ports.

Recommended characteristics:

- Compose file: `compose.prod-like.yml` or Compose profile `prod-like`;
- application source copied into immutable images;
- `frontend` runs `npm run build` at image build time and `npm run start` at runtime;
- `backend` runs optimized Composer autoload and Laravel caches where safe;
- only nginx exposes external ports;
- `db` and `redis` are internal services with named volumes;
- secrets are provided via `.env.docker.local`, Docker secrets, CI variables, or deployment platform secrets, not committed files;
- migrations are executed as a one-off release command, not as an implicit container boot side effect.

Production-like release order:

1. build backend image;
2. build frontend image;
3. build future admin image when available;
4. start `db` and `redis`;
5. run one-off backend release commands:
   - `php artisan migrate --force`;
   - `php artisan storage:link` if needed;
   - `php artisan config:cache`, `route:cache`, `view:cache` only after verifying compatibility;
6. start `backend`, `queue-worker`, `scheduler`, `reverb`;
7. start `frontend`, `admin`, `nginx`;
8. verify HTTP API, frontend rendering and websocket connection.

## Compose profiles recommendation

Use profiles to avoid starting optional services unintentionally:

| Profile | Services |
|---|---|
| default | `db`, `redis`, `backend`, `frontend`, `nginx` |
| `workers` | `queue-worker`, `scheduler` |
| `realtime` | `reverb` |
| `admin` | future `admin` |
| `prod-like` | production-like commands/build targets |

## Networks and volumes

Networks:

```text
aniyume-public   # nginx-facing services if separation is needed
aniyume-internal # backend/db/redis/workers/reverb private traffic
```

For a first implementation, one default Compose network is acceptable. Split networks later if security requirements demand it.

Named volumes:

```text
db-data
redis-data
backend-storage
backend-bootstrap-cache
frontend-node-modules     # dev only
frontend-next-cache       # dev only
composer-cache            # dev/build optimization
```

Volume policy:

- database volume must be persistent;
- Redis persistence depends on whether queues must survive restarts;
- backend `storage` must persist uploaded/generated files and logs if logs are not redirected to stdout;
- node/composer cache volumes are disposable developer convenience volumes.

## Environment file strategy

Recommended files for later implementation:

```text
.env.docker.example        # committed, no secrets
.env.docker.local          # local developer values, gitignored
.env.docker.prod.example   # committed production-like template, no secrets
```

Do not reuse a real application `.env` with secrets as a committed Docker env file.

Minimum env groups to document in examples:

- Laravel app identity: `APP_NAME`, `APP_ENV`, `APP_KEY`, `APP_DEBUG`, `APP_URL`;
- database: `DB_CONNECTION`, `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`;
- Redis: `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`;
- queues/cache/session: `QUEUE_CONNECTION`, `CACHE_STORE`, `SESSION_DRIVER`;
- Reverb backend: `BROADCAST_CONNECTION`, `REVERB_APP_ID`, `REVERB_APP_KEY`, `REVERB_APP_SECRET`, `REVERB_HOST`, `REVERB_PORT`, `REVERB_SCHEME`;
- frontend public realtime: `NEXT_PUBLIC_REVERB_APP_KEY`, `NEXT_PUBLIC_REVERB_HOST`, `NEXT_PUBLIC_REVERB_PORT`, `NEXT_PUBLIC_REVERB_SCHEME`;
- frontend API proxy: `BACKEND_URL`;
- future admin API: `ADMIN_BACKEND_URL` or framework-specific equivalent.

## Healthcheck matrix

| Service | Healthcheck | Notes |
|---|---|---|
| `db` | `pg_isready -U aniyume -d aniyume` | Required before migrations and app traffic. |
| `redis` | `redis-cli ping` | Required before Redis queues/cache. |
| `backend` | `curl -fsS http://localhost:8000/up` or lightweight API endpoint | Prefer adding a dedicated health endpoint later if missing. |
| `frontend` | `curl -fsS http://localhost:3000/` | Confirms Next.js HTTP process. |
| `admin` | `curl -fsS http://localhost:3001/` | Only when future admin is implemented. |
| `queue-worker` | process check or queue monitor | Avoid false failures from strict queue depth limits. |
| `scheduler` | `php artisan schedule:list` or process check | `schedule:list` validates app boot, not actual cron execution. |
| `reverb` | process check, optional websocket smoke test | WebSocket check may be added in integration tests. |
| `nginx` | `nginx -t` + `curl -fsS http://localhost/` | Also verify websocket upgrade manually/in tests. |

## Practical implementation plan for next Docker phase

Suggested file creation order:

1. `docs/infra-docker-design-package.md` — this design baseline.
2. `.dockerignore` files for backend/frontend contexts.
3. `infra/docker/backend/Dockerfile` or `aniyume-backend/Dockerfile` after ownership is confirmed.
4. `infra/docker/frontend/Dockerfile` or `aniyume/Dockerfile` after ownership is confirmed.
5. Optional future `infra/docker/admin/Dockerfile` only after admin runtime is clarified.
6. `infra/docker/nginx/default.conf` with frontend/backend/reverb routes.
7. `.env.docker.example` and `.env.docker.prod.example` without secrets.
8. `compose.yml` with core services.
9. `compose.dev.yml` or override file for bind mounts and direct ports.
10. `compose.prod-like.yml` or profiles for production-like build/run commands.
11. One-off command documentation for migrations, storage link, cache warmup and seeders.
12. Smoke-test checklist in docs or CI scripts.

Suggested validation order:

```bash
docker compose config
docker compose up -d db redis
docker compose run --rm backend php artisan migrate --pretend
docker compose up -d backend frontend reverb nginx
docker compose --profile workers up -d queue-worker scheduler
docker compose ps
```

Then verify:

- frontend home page through nginx;
- backend public API through nginx and, if exposed, direct backend port;
- login/register flow if test data exists;
- Watch Party websocket handshake through nginx;
- queue worker can process a test job;
- scheduler can list configured tasks.

## Risks and open decisions

| Area | Risk / decision | Recommendation |
|---|---|---|
| Admin runtime | Current docs mention Laravel admin web routes, but task requests future admin container. | Keep admin service optional until runtime is confirmed. |
| API path rewrites | nginx `/api/*`, Next `/api/external/*`, and Laravel `/api/v1` can conflict if rewritten incorrectly. | Preserve existing paths; document any rewrite before implementation. |
| Health endpoint | A stable lightweight backend health endpoint may not exist. | Add dedicated endpoint in a separate backend task, not during docs-only Docker design. |
| Migrations on startup | Automatic migrations can corrupt dev/prod-like environments unexpectedly. | Use explicit one-off migration commands. |
| Redis queue switch | Existing deployment docs mention `QUEUE_CONNECTION=database`; Docker design recommends Redis for production-like. | Confirm queue backend before implementation; keep database queue fallback compatible if needed. |
| Reverb external host/port | Container-internal `reverb:8080` differs from browser-facing host/port. | Separate backend Reverb env from `NEXT_PUBLIC_REVERB_*` values. |
| Windows bind mounts | Project path is on Windows; file watching and permissions may behave differently. | Prefer polling/HMR options only if needed; do not hardcode OS-specific behavior. |
| Secrets | Docker env examples may accidentally include real `.env` values. | Commit only example files with placeholders; gitignore local env files. |
| Storage permissions | Laravel `storage` and `bootstrap/cache` need write access. | Set container user/permissions explicitly during Dockerfile phase. |
| Build context size | Copying whole repository into Docker context can be slow and leak files. | Add focused `.dockerignore` before image implementation. |

## Rollback strategy for future implementation

If Docker phase causes local or CI regressions:

1. stop Compose services without deleting persistent volumes:
   ```bash
   docker compose down
   ```
2. keep non-Docker local workflow documented in `docs/deployment.md` as the fallback;
3. if database schema was changed by migrations, use normal application migration rollback policy rather than deleting volumes blindly;
4. remove or disable new Compose profiles first, then Dockerfiles/configs if necessary;
5. never delete named database volumes unless the environment is explicitly disposable.

## Checks for this design phase

No Docker/build/test commands were run for this docs-only task. The expected verification is documentation review against:

- `docs/deployment.md`;
- `docs/architecture-overview.md`;
- current requirement to avoid creating Dockerfile/compose files in this phase.
