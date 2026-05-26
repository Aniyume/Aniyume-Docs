# Deployment guide

## Overview

Aniyume consists of two applications:

- `aniyume` — Next.js frontend;
- `aniyume-backend` — Laravel backend.

Production deployment requires:

- web server / reverse proxy;
- PHP 8.2+ runtime;
- Composer dependencies;
- Node.js dependencies;
- database;
- Laravel queue worker;
- Laravel scheduler;
- Laravel Reverb websocket server;
- storage symlink.

## Backend deployment

### Install dependencies

```bash
cd aniyume-backend
composer install --no-dev --optimize-autoloader
npm install
npm run build
```

### Environment

Required `.env` groups:

```env
APP_NAME=Aniyume
APP_ENV=production
APP_KEY=base64:...
APP_DEBUG=false
APP_URL=https://api.example.com

DB_CONNECTION=pgsql
DB_HOST=...
DB_PORT=5432
DB_DATABASE=...
DB_USERNAME=...
DB_PASSWORD=...

QUEUE_CONNECTION=database

BROADCAST_CONNECTION=reverb
REVERB_APP_ID=...
REVERB_APP_KEY=...
REVERB_APP_SECRET=...
REVERB_HOST=...
REVERB_PORT=443
REVERB_SCHEME=https
```

### Migrations

Preview:

```bash
php artisan migrate --pretend
```

Apply:

```bash
php artisan migrate --force
```

Seed roles/admin data if needed:

```bash
php artisan db:seed --class=RoleSeeder --force
```

### Storage

```bash
php artisan storage:link
```

Ensure `storage/` and `bootstrap/cache/` are writable by the web server user.

### Cache optimization

```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

If routes use closures and route caching fails, either avoid `route:cache` or convert closure routes to controllers.

### Queue worker

Required for non-immediate queued jobs/imports.

Example Supervisor program:

```ini
[program:aniyume-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/aniyume-backend/artisan queue:work --sleep=3 --tries=3 --timeout=300
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=1
redirect_stderr=true
stdout_logfile=/var/log/aniyume-worker.log
```

### Scheduler

Add cron:

```cron
* * * * * cd /var/www/aniyume-backend && php artisan schedule:run >> /dev/null 2>&1
```

Verify:

```bash
php artisan schedule:list
```

Expected tasks:

```text
import:anime
import:episodes --limit=200
episodes:sync-ongoing --days=1
```

### Reverb

Run Reverb:

```bash
php artisan reverb:start
```

In production use Supervisor/systemd.

Reverse proxy must support websocket upgrade.

Nginx example:

```nginx
location /app/ {
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
    proxy_pass http://127.0.0.1:8080;
}
```

## Frontend deployment

### Install and build

```bash
cd aniyume
npm install
npm run build
```

### Environment

Example:

```env
BACKEND_URL=https://api.example.com/api/v1

NEXT_PUBLIC_REVERB_APP_KEY=...
NEXT_PUBLIC_REVERB_HOST=ws.example.com
NEXT_PUBLIC_REVERB_PORT=443
NEXT_PUBLIC_REVERB_SCHEME=https
```

### Run

```bash
npm run start
```

Use process manager:

- PM2;
- systemd;
- Docker;
- platform runtime.

## Reverse proxy layout

Recommended production layout:

```text
https://example.com          -> Next.js frontend
https://api.example.com      -> Laravel backend
wss://ws.example.com         -> Laravel Reverb
```

Alternative:

```text
https://example.com          -> Next.js
https://example.com/api      -> Laravel API reverse proxy
https://example.com/app      -> Reverb websocket
```

## Health checks

Backend:

```text
GET /up
GET /api/v1/public/anime?per_page=1
```

Frontend:

```text
GET /
GET /catalog
```

Realtime:

- create Watch Party;
- join presence channel;
- send chat message;
- verify player sync.

## Deployment checklist

Backend:

- [ ] `.env` configured;
- [ ] `APP_DEBUG=false`;
- [ ] dependencies installed;
- [ ] migrations applied;
- [ ] roles seeded;
- [ ] storage linked;
- [ ] queue worker running;
- [ ] scheduler cron installed;
- [ ] Reverb running;
- [ ] CORS origins configured;
- [ ] admin account verified.

Frontend:

- [ ] `.env` configured;
- [ ] build succeeds;
- [ ] backend proxy works;
- [ ] Reverb env matches backend;
- [ ] CSP allows required media/ws hosts.

Validation:

- [ ] login/register;
- [ ] anime catalog;
- [ ] anime page/player;
- [ ] watch history;
- [ ] favorites/list status;
- [ ] comments/ratings;
- [ ] watch party;
- [ ] admin panel;
- [ ] imports/scheduler.

## Rollback notes

Application rollback:

1. deploy previous release;
2. restore previous `.env` if changed;
3. restart queue/reverb/frontend;
4. clear caches.

Database rollback:

```bash
php artisan migrate:rollback --step=1
```

For destructive migrations, rollback restores structure only if data was dropped. Always create DB backup before destructive migrations.
