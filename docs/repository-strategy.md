# Aniyume repository strategy

## Краткое решение

Aniyume следует оформить как набор независимых application repositories под одной GitHub Organization:

```text
GitHub Organization: Aniyume

Repositories:
├── aniyume-web          # Aniyume Web, пользовательский Next.js frontend
├── aniyume-api          # Aniyume API, Laravel backend/API/workers/realtime
└── aniyume-admin-web    # Aniyume Admin Web, admin Next.js frontend
```

Во всех repositories:

```text
default branch: dev
production branch: release
```

## Repo naming

| Display name | Repo name | Назначение |
|---|---|---|
| Aniyume Web | `aniyume-web` | Публичный пользовательский frontend |
| Aniyume API | `aniyume-api` | Backend API, domain logic, DB migrations, queues, scheduler, realtime |
| Aniyume Admin Web | `aniyume-admin-web` | Административная панель |

Почему не `frontend`, `backend`, `admin`:

- имена должны быть понятны вне локального контекста;
- `aniyume-api` лучше отражает публичный контракт, чем `backend`;
- `aniyume-admin-web` сразу показывает, что это web UI, а не API/admin service.

## Ownership

### `aniyume-web`

Владеет пользовательским UI:

- Next.js app;
- components/contexts/hooks/lib;
- публичные assets;
- frontend Dockerfile;
- frontend-specific README/docs.

Не владеет:

- API contract как source of truth;
- DB migrations;
- production compose целиком;
- secrets.

### `aniyume-api`

Владеет backend и API contract:

- Laravel app;
- routes/controllers/services/models;
- database migrations/seeders/factories;
- tests;
- queues/scheduler/reverb runtime logic;
- backend Dockerfile;
- API/deployment/security docs как временный canonical source.

Не владеет:

- пользовательским Next.js UI;
- admin UI implementation;
- frontend build artifacts.

### `aniyume-admin-web`

Владеет admin UI:

- Next.js admin scaffold/application;
- admin routes/pages/components;
- admin-specific env example;
- admin Dockerfile после появления;
- admin user docs.

Не владеет:

- admin API endpoints;
- RBAC/domain permissions на backend;
- shared production orchestration.

## Branch model

```text
feature/* → dev → release → production
hotfix/*  → release → dev
```

### `dev`

Назначение:

- ежедневная интеграция;
- тестовый/staging deploy;
- проверка совместимости web/api/admin;
- default branch в GitHub.

Правила:

- merge только через PR;
- CI обязателен;
- допускаются незавершенные фичи только если они скрыты feature flag или не ломают тестовый стенд.

### `release`

Назначение:

- production-ready код;
- источник production Docker images/deploy;
- стабильная линия для hotfix.

Правила:

- только PR из `dev` или `hotfix/*`;
- обязательно green CI;
- обязательно review/approval;
- deploy только после approval GitHub Environment `production`;
- direct push запрещен.

## Deploy model

### Development/staging

Триггер:

- merge в `dev`.

Действия:

- build/test;
- optional docker image publish с тегом `dev-<sha>`;
- deploy на test/staging host.

### Production

Триггер:

- merge в `release` или вручную approved workflow на `release`.

Действия:

1. CI запускает тесты.
2. Собираются production Docker images.
3. Images публикуются в GHCR:
   - `ghcr.io/aniyume/aniyume-web:<sha>`;
   - `ghcr.io/aniyume/aniyume-api:<sha>`;
   - `ghcr.io/aniyume/aniyume-admin-web:<sha>`.
4. Production environment требует approval.
5. Docker host подтягивает images и перезапускает services.
6. Healthchecks подтверждают успешный deploy.

Принцип: production host не должен быть местом сборки исходников. Он должен запускать уже собранные images.

## Docs strategy

### Практичный старт

Сделать `aniyume-api/docs` временным canonical source для общей документации:

- architecture;
- API contract;
- data model;
- deployment;
- security;
- testing strategy;
- admin/API design docs.

В `aniyume-web` и `aniyume-admin-web` оставить:

- README с локальными командами;
- docs только по frontend/admin-specific решениям;
- ссылки на canonical docs.

### Целевое состояние

Если документация активно растет, создать отдельный repository:

```text
aniyume-docs
```

Туда вынести cross-repo документы. Application repos оставляют только technical quickstart и component-specific docs.

## Infra/orchestration strategy

Текущий root `docker-compose.yml` завязан на относительные пути монорепо:

```text
./aniyume
./aniyume-backend
./infra/docker/nginx/default.conf
```

После разделения на repositories такая схема неудобна для production.

Рекомендуемая цель:

```text
aniyume-infra
├── compose/
│   ├── docker-compose.release.yml
│   └── .env.example
├── nginx/
├── scripts/
└── runbooks/
```

Если строго нужны только три repo, временно хранить deploy compose/runbooks в `aniyume-api/infra`, потому что API repo ближе всего к DB/Redis/queues/migrations. Но это компромисс, не финальная архитектура.

## Migration safety checklist

Перед переносом каждого repo:

- [ ] выбран canonical source;
- [ ] все разработчики сообщили о локальных незапушенных изменениях;
- [ ] проверены текущие branches/tags;
- [ ] проверены secrets и `.env`;
- [ ] build artifacts исключены;
- [ ] создан backup branch/tag;
- [ ] создан пустой target GitHub repo;
- [ ] история перенесена через git, не ручным копированием;
- [ ] создан `dev`;
- [ ] создан `release` от стабильного commit;
- [ ] включен branch protection;
- [ ] настроены CI checks.

## Минимальный порядок внедрения

1. Organization + permissions.
2. `aniyume-api` migration, потому что API является центром домена и deploy.
3. `aniyume-web` migration, с отдельным разбором diverged frontend history.
4. `aniyume-admin-web` initial import.
5. Branch protection и CI во всех repos.
6. Staging deploy из `dev`.
7. Production deploy из `release`.
8. Решение по `aniyume-infra`/canonical docs после первой стабильной production выкладки.
