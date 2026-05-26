# Architecture overview

## Назначение проекта

Aniyume — web-платформа для просмотра и учета anime-контента.

Ключевые возможности:

- каталог anime с фильтрацией, поиском и сортировкой;
- страницы anime с эпизодами и источниками плеера;
- пользовательская авторизация;
- избранное и списки просмотра;
- история просмотра и прогресс;
- рейтинги и комментарии;
- друзья;
- Watch Party с realtime-синхронизацией плеера и чатом;
- admin panel для управления anime, episodes, tags, users, comments и import jobs.

## Общая схема

```text
Browser
  |
  | HTTP / WebSocket
  v
Next.js frontend (aniyume)
  |
  | /api/external/* proxy
  v
Laravel backend (aniyume-backend)
  |
  | Eloquent / queues / schedule / broadcast
  v
Database + Reverb + external anime/video APIs
```

## Frontend

Технологии:

- Next.js 16;
- React 19;
- TypeScript;
- App Router;
- Tailwind CSS;
- Laravel Echo + Pusher protocol for Reverb;
- HLS/P2P video player.

### API access

Frontend использует централизованный proxy:

- frontend route: `aniyume/app/api/external/[...path]/route.ts`;
- frontend client: `aniyume/lib/api.ts`;
- backend base: `BACKEND_URL`, по умолчанию `http://127.0.0.1:8000/api/v1`.

Proxy отвечает за:

- добавление `/public` для публичных backend endpoints;
- проброс Bearer token;
- нормализацию auth headers;
- единый frontend-facing API path `/api/external/*`.

### Public API examples

Frontend:

```text
/api/external/anime
/api/external/public/anime
/api/external/schedule
```

Backend target:

```text
/api/v1/public/anime
/api/v1/public/schedule
```

### Private API examples

Frontend:

```text
/api/external/profile/me
/api/external/watch-history
/api/external/watch-party/{code}
```

Backend target:

```text
/api/v1/profile/me
/api/v1/watch-history
/api/v1/watch-party/{code}
```

## Backend

Технологии:

- PHP 8.2+;
- Laravel 12;
- Laravel Sanctum;
- Laravel Reverb;
- Eloquent ORM;
- Scheduler;
- Queue jobs;
- Scribe/Swagger documentation packages.

### Backend module groups

Основные API modules:

- `AuthController` — регистрация, вход, текущий пользователь, logout;
- `AnimeController` — публичный каталог, details, recommendations, community stats, banners;
- `EpisodeController` — эпизоды и player sources;
- `TagController` — tags/genres list;
- `UserAnimeListController` — список просмотра и статус anime у пользователя;
- `WatchHistoryController` — история просмотра и прогресс;
- `FavoritesController` — избранное;
- `RatingsController` — пользовательские рейтинги;
- `CommentsController` — комментарии;
- `UserProfileController` — профиль;
- `UserStatisticsController` — статистика;
- `FriendshipController` — друзья и заявки;
- `WatchPartyController` — комнаты совместного просмотра;
- `PaymentController` — premium subscription endpoint.

Admin modules:

- `AnimeManagementController`;
- `EpisodeManagementController`;
- `TagManagementController`;
- `UserManagementController`;
- `CommentModerationController`;
- `ImportManagementController`;
- `AuditLogController`;
- `DashboardController`.

### Routing

Основной API prefix:

```text
/api/v1
```

Public group:

```text
/api/v1/public/*
```

Protected group:

```php
Route::middleware('auth:sanctum')->group(...)
```

Admin web routes:

```php
Route::middleware(['auth', 'admin'])->group(...)
```

### Auth model

Проект использует Sanctum Bearer tokens:

1. frontend получает token при login/register;
2. token хранится в `localStorage` как `userToken`;
3. API client/proxy отправляет `Authorization: Bearer <token>`;
4. backend защищает private endpoints через `auth:sanctum`.

### Realtime

Watch Party realtime построен на:

- Laravel Reverb;
- Laravel Echo на frontend;
- presence channel `watch-party.{roomCode}`;
- private channel `user.{userId}` для invite events.

Основные events:

- `PlayerSyncEvent` — синхронизация плеера;
- `ChatMessageEvent` — live chat;
- `RoomClosedEvent` — закрытие комнаты;
- `FriendInviteEvent` — invite друга.

Realtime events используют `ShouldBroadcastNow`, чтобы не зависеть от queue worker для UX-critical событий.

### Scheduler

Актуальное место schedule config:

```text
aniyume-backend/routes/console.php
```

Активные задачи:

```text
0 3   * * *  php artisan import:anime
0 *   * * *  php artisan import:episodes --limit=200
0 */6 * * *  php artisan episodes:sync-ongoing --days=1
```

Legacy `app/Console/Kernel.php` оставлен только для совместимости.

### Import pipeline

Канонический anime import:

```text
ImportAnimeCommand -> ShikimoriImportService
```

Канонический episode import:

```text
ImportEpisodesCommand -> EpisodeImportService
```

Backward-compatible alias:

```text
episodes:import
```

но новые scheduler/jobs должны использовать:

```text
import:episodes
```

## Data model overview

Основные модели:

- `Anime`;
- `Episode`;
- `Tag`;
- `User`;
- `Role`;
- `Favorite`;
- `Rating`;
- `Comment`;
- `WatchHistory`;
- `Friendship`;
- `WatchPartyRoom`;
- `WatchPartyParticipant`;
- `WatchPartyMessage`;
- `ImportLog`;
- `AuditLog`;
- `BlacklistedAnime`.

Теги/жанры представлены через:

```text
tags
anime_tag
```

Отдельные `Genre`/`Studio` models удалены как неиспользуемые.

## Quality gates

Минимальный набор проверок перед сдачей/деплоем:

Frontend:

```bash
npm run build
```

Backend:

```bash
php artisan test
php artisan route:list
php artisan schedule:list
```

PHP syntax for changed files:

```bash
php -l path/to/file.php
```

Migration safety:

```bash
php artisan migrate --pretend
```

## Дипломная ценность архитектуры

Сильные стороны проекта для дипломной работы:

- fullstack architecture: Next.js + Laravel;
- REST API + realtime WebSocket слой;
- auth/roles/admin panel;
- background imports и scheduler;
- external API integrations;
- video streaming/P2P layer;
- documentation-driven cleanup;
- clear separation of public/private API;
- production concerns: CORS, proxy, Sanctum, Reverb, queues, migrations.

## Следующие улучшения

Чтобы проект выглядел еще сильнее как диплом:

1. добавить ERD/data model diagram;
2. добавить sequence diagrams для login, watch history, watch party;
3. добавить API contract document;
4. добавить deployment guide;
5. добавить security notes;
6. добавить тесты на ключевые API endpoints;
7. добавить health checks и observability notes.
