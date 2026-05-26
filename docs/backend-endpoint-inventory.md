# Backend endpoint inventory

## Назначение

Этот документ фиксирует текущее состояние backend endpoints в `aniyume-backend` до начала крупного рефакторинга.

Baseline сверялся с `routes/api.php`, `routes/web.php`, `routes/channels.php` и `php artisan route:list`. README содержит только краткое описание; полный контракт по маршрутам фиксируется здесь.

Цели:

- иметь baseline по всем API и web/admin маршрутам;
- понимать, что нельзя сломать во время рефакторинга;
- разделить публичные, приватные, realtime и admin маршруты;
- подготовить основу для выноса Blade-admin в отдельный Next.js admin client.

---

## 1. API prefix

Основной API prefix:

```text
/api/v1
```

Защищенные API endpoints используют:

```text
auth:sanctum
```

Broadcast routes также подключены внутри `v1` группы.

Дополнительно framework route `broadcasting/auth` присутствует без `/api/v1` prefix из broadcast service provider/framework registration. Для frontend/watch party важно явно проверять, какой auth endpoint используется клиентом.

---

## 2. System / utility endpoints

| Method | Path | Controller/Handler | Auth | Notes |
|---|---|---|---|---|
| POST | `/api/v1/trap` | Closure | No | Honeypot endpoint для ботов |
| POST | `/api/v1/broadcasting/auth` | `Broadcast::routes()` | Yes | Авторизация private/presence каналов |
| GET/POST | `/broadcasting/auth` | framework broadcast route | Yes | Дополнительный broadcast auth route без `/api/v1`, виден в `route:list` |
| GET | `/up` | Laravel health route | No | Framework health check route |
| GET | `/sanctum/csrf-cookie` | Sanctum | No | CSRF cookie для cookie-based Sanctum сценариев |
| GET | `/storage/{path}` | Laravel storage route | No | Public storage files |

---

## 3. Public API endpoints

Префикс группы:

```text
/api/v1/public
```

| Method | Path | Controller | Action | Notes |
|---|---|---|---|---|
| GET | `/api/v1/public/anime` | `AnimeController` | `index` | Каталог anime, фильтры, поиск, сортировка |
| GET | `/api/v1/public/schedule` | `ScheduleController` | `index` | Публичное расписание |
| GET | `/api/v1/public/episodes/translators` | `EpisodeController` | `getAllTranslators` | Список переводчиков |
| GET | `/api/v1/public/episodes` | `EpisodeController` | `index` | Поиск/список эпизодов |
| GET | `/api/v1/public/tags` | `TagController` | `index` | Список тегов/жанров |
| GET | `/api/v1/public/anime/{anime}` | `AnimeController` | `show` | Детальная страница anime |
| GET | `/api/v1/public/anime/{anime}/banner` | `AnimeController` | `getBanner` | Баннер/cover |
| GET | `/api/v1/public/anime/{anime}/comments` | `CommentsController` | `index` | Одобренные комментарии |
| GET | `/api/v1/public/anime/{anime}/episodes` | `EpisodeController` | `getByAnime` | Список эпизодов anime |
| GET | `/api/v1/public/anime/{anime}/episodes/{episodeNumber}/sources` | `EpisodeController` | `getPlayerSources` | Источники плеера для эпизода |
| GET | `/api/v1/public/anime/{anime}/community-stats` | `AnimeController` | `getCommunityStats` | Распределение пользовательских статусов |
| GET | `/api/v1/public/anime/{anime}/recommendations` | `AnimeController` | `getRecommendations` | Связанные и похожие anime |
| GET | `/api/v1/public/episodes/{episode}` | `EpisodeController` | `show` | Детали эпизода |
| GET | `/api/v1/public/episodes/{episode}/player` | `EpisodeController` | `getPlayer` | Player endpoint |
| GET | `/api/v1/public/users/{userId}/statistics` | `UserStatisticsController` | `getStatistics` | Публичная/полупубличная статистика пользователя |

### Public API criticality

**High criticality** — нельзя ломать в первых фазах:
- `/public/anime`
- `/public/anime/{anime}`
- `/public/anime/{anime}/episodes`
- `/public/anime/{anime}/episodes/{episodeNumber}/sources`
- `/public/tags`

**Medium criticality**:
- `/public/anime/{anime}/recommendations`
- `/public/anime/{anime}/community-stats`
- `/public/episodes`
- `/public/schedule`

---

## 4. Authentication endpoints

| Method | Path | Controller | Action | Auth | Notes |
|---|---|---|---|---|---|
| POST | `/api/v1/auth/register` | `AuthController` | `register` | No | Регистрация |
| POST | `/api/v1/auth/login` | `AuthController` | `login` | No | Логин, выдача токена |
| GET | `/api/v1/user` | `AuthController` | `me` | Yes | Текущий пользователь |
| POST | `/api/v1/auth/logout` | `AuthController` | `logout` | Yes | Logout текущего токена |

### Auth criticality

**Very high** — это базовый вход в систему.

---

## 5. Authenticated user content endpoints

### Comments

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/my-comments` | `CommentsController` | `userComments` |
| POST | `/api/v1/comments` | `CommentsController` | `store` |
| PUT/PATCH | `/api/v1/comments/{comment}` | `CommentsController` | `update` |
| DELETE | `/api/v1/comments/{comment}` | `CommentsController` | `destroy` |

### Profile

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/profile/me` | `UserProfileController` | `getFullProfile` |
| PUT | `/api/v1/profile/me` | `UserProfileController` | `update` |
| POST | `/api/v1/profile/me/avatar` | `UserProfileController` | `uploadAvatar` |

Примечание: старые/неточные варианты `/api/v1/profile/update` и `/api/v1/profile/avatar` не являются текущими routes.

### Statistics

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/statistics/me` | `UserStatisticsController` | `getStatistics` |
| GET | `/api/v1/statistics/me/episodes-summary` | `UserStatisticsController` | `getEpisodesSummary` |

### User anime list

| Method | Path | Controller | Action |
|---|---|---|---|
| POST | `/api/v1/anime/{anime}/status` | `UserAnimeListController` | `updateStatus` |
| GET | `/api/v1/anime/{anime}/user-status` | `UserAnimeListController` | `getUserStatus` |
| PATCH | `/api/v1/anime/{anime}/episodes-watched/{episodesWatched}` | `UserAnimeListController` | `updateEpisodesWatched` |
| GET | `/api/v1/my-anime-list/{status?}` | `UserAnimeListController` | `getList` |

### Favorites

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/favorites` | `FavoritesController` | `index` |
| POST | `/api/v1/favorites` | `FavoritesController` | `store` |
| DELETE | `/api/v1/favorites/{animeId}` | `FavoritesController` | `destroy` |
| GET | `/api/v1/favorites/{animeId}/check` | `FavoritesController` | `checkFavorite` |

### Watch history

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/watch-history` | `WatchHistoryController` | `index` |
| POST | `/api/v1/watch-history` | `WatchHistoryController` | `store` |
| GET | `/api/v1/watch-history/{id}` | `WatchHistoryController` | `show` |
| DELETE | `/api/v1/watch-history/{id}` | `WatchHistoryController` | `destroy` |
| GET | `/api/v1/watch-history/anime/{animeId}/history` | `WatchHistoryController` | `getByAnime` |
| GET | `/api/v1/watch-history/anime/{animeId}/last-episode` | `WatchHistoryController` | `getLastWatchedEpisode` |

### Ratings

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/api/v1/ratings` | `RatingsController` | `index` |
| POST | `/api/v1/ratings` | `RatingsController` | `store` |
| DELETE | `/api/v1/ratings/{rating}` | `RatingsController` | `destroy` |
| GET | `/api/v1/ratings/anime/{animeId}` | `RatingsController` | `getUserRating` |

### Payment

| Method | Path | Controller | Action |
|---|---|---|---|
| POST | `/api/v1/payment/premium` | `PaymentController` | `subscribeToPremium` |

### User content criticality

**High criticality**:
- profile
- favorites
- user anime list
- watch history
- comments
- ratings

---

## 6. Social endpoints

### Friendship

| Method | Path | Controller | Action | Notes |
|---|---|---|---|---|
| GET | `/api/v1/friends` | `FriendshipController` | `index` | Список друзей |
| GET | `/api/v1/friends/requests` | `FriendshipController` | `requests` | Входящие/исходящие заявки |
| GET | `/api/v1/friends/requests/count` | `FriendshipController` | `requestsCount` | Badge counter |
| POST | `/api/v1/friends/{userId}` | `FriendshipController` | `send` | Отправить заявку |
| POST | `/api/v1/friends/{userId}/accept` | `FriendshipController` | `accept` | Принять заявку |
| POST | `/api/v1/friends/{userId}/decline` | `FriendshipController` | `decline` | Отклонить/удалить |
| GET | `/api/v1/friends/{userId}/status` | `FriendshipController` | `status` | Проверка статуса |
| GET | `/api/v1/users/search` | `FriendshipController` | `search` | Поиск пользователей |

### Watch Party

| Method | Path | Controller | Action | Notes |
|---|---|---|---|---|
| POST | `/api/v1/watch-party` | `WatchPartyController` | `create` | Создать комнату |
| GET | `/api/v1/watch-party/{code}` | `WatchPartyController` | `show` | Информация о комнате |
| POST | `/api/v1/watch-party/{code}/join` | `WatchPartyController` | `join` | Войти |
| POST | `/api/v1/watch-party/{code}/leave` | `WatchPartyController` | `leave` | Выйти |
| POST | `/api/v1/watch-party/{code}/sync` | `WatchPartyController` | `sync` | Синхронизация плеера хостом |
| POST | `/api/v1/watch-party/{code}/message` | `WatchPartyController` | `sendMessage` | Сообщение в чат |
| GET | `/api/v1/watch-party/{code}/messages` | `WatchPartyController` | `getMessages` | История чата |
| POST | `/api/v1/watch-party/{code}/invite` | `WatchPartyController` | `invite` | Пригласить друга |
| DELETE | `/api/v1/watch-party/{code}` | `WatchPartyController` | `close` | Закрыть комнату |

### Social criticality

**High criticality**:
- watch party create/show/join/sync/message
- friendship send/accept/status

---

## 7. Broadcast channels

Файл: `routes/channels.php`

| Channel | Type | Access rule | Purpose |
|---|---|---|---|
| `App.Models.User.{id}` | Private | Только сам пользователь | Системный Sanctum channel |
| `user.{userId}` | Private | Только сам пользователь | Приглашения в комнату |
| `watch-party.{code}` | Presence | Только активные участники комнаты | Presence и realtime Watch Party |

### Broadcast criticality

**Very high** для watch party demo и realtime сценариев.

---

## 8. Web/admin routes

Эти маршруты не являются публичным пользовательским API. Это встроенная Blade-admin панель.

### Root / misc web

| Method | Path | Handler | Notes |
|---|---|---|---|
| GET | `/` | closure redirect | Redirect на `/admin/login` |
| GET | `/login` | closure | JSON 401 fallback |

### Admin auth

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/login` | `Admin\AuthController` | `showLoginForm` |
| POST | `/admin/login` | `Admin\AuthController` | `login` |
| GET | `/admin/logout` | closure/view | confirm logout page |
| POST | `/admin/logout` | `Admin\AuthController` | `logout` |

### Admin dashboard

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/dashboard` | `DashboardController` | `index` |

### Admin users

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/users` | `UserManagementController` | `index` |
| POST | `/admin/users` | `UserManagementController` | `store` | route resource, наличие UI/use case требует отдельной проверки |
| GET | `/admin/users/create` | `UserManagementController` | `create` | route resource, наличие UI/use case требует отдельной проверки |
| GET | `/admin/users/{user}` | `UserManagementController` | `show` |
| GET | `/admin/users/{user}/edit` | `UserManagementController` | `edit` | route resource, наличие UI/use case требует отдельной проверки |
| PUT/PATCH | `/admin/users/{user}` | `UserManagementController` | `update` | route resource, наличие UI/use case требует отдельной проверки |
| POST | `/admin/users/{user}/ban` | `UserManagementController` | `ban` |
| POST | `/admin/users/{user}/unban` | `UserManagementController` | `unban` |
| DELETE | `/admin/users/{user}` | `UserManagementController` | `destroy` |

Примечание: `Route::resource('users', ...)` регистрирует полный набор resource routes. Если часть экранов/методов фактически не используется, это нужно уточнять при admin migration, а не удалять на этапе baseline.

### Admin anime

| Method | Path | Controller | Action |
|---|---|---|---|
| POST | `/admin/anime/bulk-delete` | `AnimeManagementController` | `bulkDestroy` |
| GET | `/admin/anime` | `AnimeManagementController` | `index` |
| GET | `/admin/anime/create` | `AnimeManagementController` | `create` |
| POST | `/admin/anime` | `AnimeManagementController` | `store` |
| GET | `/admin/anime/{anime}` | `AnimeManagementController` | `show` |
| GET | `/admin/anime/{anime}/edit` | `AnimeManagementController` | `edit` |
| PUT/PATCH | `/admin/anime/{anime}` | `AnimeManagementController` | `update` |
| DELETE | `/admin/anime/{anime}` | `AnimeManagementController` | `destroy` |

### Admin tags

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/tags` | `TagManagementController` | `index` |
| GET | `/admin/tags/create` | `TagManagementController` | `create` |
| POST | `/admin/tags` | `TagManagementController` | `store` |
| GET | `/admin/tags/{tag}/edit` | `TagManagementController` | `edit` |
| PUT/PATCH | `/admin/tags/{tag}` | `TagManagementController` | `update` |
| DELETE | `/admin/tags/{tag}` | `TagManagementController` | `destroy` |

### Admin episodes

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/episodes` | `EpisodeManagementController` | `index` |
| GET | `/admin/episodes/{episode}/edit` | `EpisodeManagementController` | `edit` |
| PUT | `/admin/episodes/{episode}` | `EpisodeManagementController` | `update` |
| POST | `/admin/episodes/import-all` | `EpisodeManagementController` | `importAll` |
| POST | `/admin/episodes/{anime}/import` | `EpisodeManagementController` | `importForAnime` |
| POST | `/admin/episodes/bulk-import` | `EpisodeManagementController` | `bulkImport` |
| DELETE | `/admin/episodes/{id}` | `EpisodeManagementController` | `destroy` |

### Admin comments moderation

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/comments` | `CommentModerationController` | `index` |
| POST | `/admin/comments/{comment}/approve` | `CommentModerationController` | `approve` |
| POST | `/admin/comments/{comment}/reject` | `CommentModerationController` | `reject` |
| DELETE | `/admin/comments/{comment}` | `CommentModerationController` | `destroy` |

### Admin import

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/import` | `ImportManagementController` | `index` |
| POST | `/admin/import/run` | `ImportManagementController` | `run` |
| GET | `/admin/import/logs` | `ImportManagementController` | `logs` |

### Admin audit logs

| Method | Path | Controller | Action |
|---|---|---|---|
| GET | `/admin/audit-logs` | `AuditLogController` | `index` |

### Admin criticality

Все admin web routes считаются **временно поддерживаемыми**, но в целевой архитектуре должны быть заменены на admin API + отдельный Next.js admin client.

### API documentation routes

В route list также присутствуют documentation routes от установленных packages:

| Method | Path | Handler | Notes |
|---|---|---|---|
| GET | `/docs` | L5 Swagger docs route | Swagger UI route |
| GET | `/docs/asset/{asset}` | L5 Swagger asset route | Swagger UI assets |
| GET | `/api/documentation` | L5 Swagger API route | Swagger/OpenAPI JSON route |
| GET | `/api/oauth2-callback` | L5 Swagger callback route | OAuth callback route |

Scribe установлен и может генерировать docs командой `php artisan scribe:generate`, но фактические web routes документации в текущем `route:list` также включают L5 Swagger. Поэтому нельзя утверждать, что `/docs` — исключительно Scribe UI без проверки конфигурации.

---

## 9. Что должно остаться после рефакторинга

### Должно остаться
- весь public API;
- весь protected user API;
- friendship API;
- watch party API;
- broadcast channels;
- payment endpoint;
- auth endpoints.

### Должно быть переработано, но сохранено функционально
- comments;
- ratings;
- watch history;
- anime catalog query logic;
- friendship and watch party internals.

### Должно быть вынесено из Blade/web слоя
- весь `/admin/*` web UI;
- admin session-based interaction;
- Blade views для admin.

---

## 10. Основные риски при рефакторинге

1. Сломать публичные anime endpoints.
2. Сломать auth/token flow.
3. Сломать watch party realtime сценарий.
4. Сломать admin импорт и moderation до появления admin API.
5. Потерять parity между старой Blade-admin и новой Next-admin.

---

## 11. Итог

Этот inventory — стартовая карта backend surface area. Перед изменением любого крупного модуля необходимо сверяться с этим документом, чтобы понимать:

- какой endpoint обслуживается;
- какая у него критичность;
- должен ли он остаться в финальной архитектуре;
- относится ли он к user API или к legacy Blade-admin слою.
