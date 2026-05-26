# Docker setup

This is the first working Docker phase for local/dev usage. It keeps the existing backend and frontend business code untouched and follows the design package in `docs/infra-docker-design-package.md`.

## Services

`docker compose up --build` starts:

- `db` — PostgreSQL 16 on container port `5432`, host port `5433` by default.
- `redis` — Redis 7 on `6379`.
- `backend` — Laravel HTTP runtime on `8000` using `php artisan serve`.
- `frontend` — Next.js dev server on `3000`.
- `queue-worker` — Laravel `queue:work` with Redis queue connection.
- `scheduler` — Laravel `schedule:work`.
- `reverb` — Laravel Reverb websocket server on `8080`.
- `nginx` — reverse proxy on host port `8088`.

The stack also contains a separate `aniyume-admin` source directory in the repository, but the current Compose stack intentionally does **not** start it as a container. For the defense/demo baseline the operational admin path is either the legacy Laravel Blade admin under `/admin/*` through nginx, or a manually started Next.js admin scaffold if its transitional API scenarios need to be shown separately.

Primary entrypoint: <http://localhost:8088>

Useful direct ports:

- Frontend: <http://localhost:3000>
- Backend health: <http://localhost:8000/up>
- Nginx health: <http://localhost:8088/nginx-health>
- Reverb direct port: `ws://localhost:8080/app/...`

## Quick start

```bash
docker compose config
docker compose up --build -d
docker compose ps
```

The dev commands check container dependency volumes and run `composer install` / `npm ci` when the named `vendor` or `node_modules` volumes are empty. This keeps Windows host dependencies from being required inside the containers.

In another terminal, run one-off Laravel setup commands when needed:

```bash
docker compose exec backend php artisan key:generate
docker compose exec backend php artisan migrate
docker compose exec backend php artisan storage:link
```

The compose file intentionally does not run key generation, migrations, storage link, seeders, or admin account creation automatically on container boot.

## Environment files

- `.env.docker.example` is committed and contains only local placeholders.
- For private local overrides, create `.env.docker.local` and pass it explicitly, for example:

```bash
docker compose --env-file .env.docker.local up --build
```

Do not commit real secrets. `APP_KEY`, Reverb keys, database passwords, mail credentials, and any API tokens must be replaced outside committed example files.

## Routing

Nginx routes:

- `/` -> `frontend:3000`
- `/api/*` -> `backend:8000` without path rewrite
- `/admin/*` -> existing Laravel Blade-admin routes in backend, without creating a new admin product
- `/app/*` -> `reverb:8080` with WebSocket upgrade headers
- `/storage/*` and `/up` -> backend

The separate `aniyume-admin` container is not enabled in this Docker phase. The repo contains an `aniyume-admin` scaffold, but it is not wired into Compose/nginx yet; `/admin/*` is routed to the existing Laravel Blade-admin routes in backend.

## Known issues and manual steps

- You must generate `APP_KEY` for a fresh Docker volume/environment before using encrypted Laravel features. For a committed-free local secret, copy `.env.docker.example` to a private env file and pass it with `--env-file`, or run `php artisan key:generate --show` and set the value outside committed examples.
- You must run migrations manually before database-backed pages/API flows work: `docker compose exec backend php artisan migrate`.
- You must create the public storage symlink manually if uploaded/public files are needed: `docker compose exec backend php artisan storage:link`.
- Run seeders and create an admin account manually according to the current backend seed/admin flow. This stack does not invent or change admin auth.
- The backend healthcheck uses Laravel's existing `/up` route. If middleware or app boot fails because setup is incomplete, the container may show unhealthy until env/key/migrations are fixed.
- Frontend dependency metadata must stay in sync: Docker uses `npm ci`, so `aniyume/package-lock.json` must match `aniyume/package.json`.
- The frontend Docker runtime uses the dev server for this phase. A `prod-like` Dockerfile target exists, but compose is intentionally optimized for reproducible local/dev startup first.
- `aniyume/next.config.ts` contains a hardcoded `/api-storage` rewrite to `127.0.0.1:8000`; this is frontend application config and was not changed in this infra-only phase. Through nginx, `/storage/*` is proxied to backend.
- Windows bind mounts can make file watching slower. Add polling-specific env only if the team observes HMR issues.

## Verified local checks

The current stack was verified on Docker Desktop with:

```bash
docker version
docker compose version
docker compose config
docker compose up --build -d
docker compose ps
curl.exe -I --max-time 60 http://localhost:3000/
curl.exe --max-time 60 http://localhost:8000/up
curl.exe --max-time 60 http://localhost:8088/nginx-health
curl.exe -I --max-time 60 http://localhost:8088/
```

Expected service state after startup: `db`, `redis`, `backend`, `frontend`, and `nginx` healthy; `queue-worker`, `scheduler`, and `reverb` running. There is no separate `admin` container in the current Compose baseline; `/admin/*` is routed to the backend.
