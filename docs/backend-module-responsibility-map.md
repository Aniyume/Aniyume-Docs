# Backend module responsibility map

## Назначение

Этот документ фиксирует текущее распределение ответственности в backend и помогает понять:

- какие модули уже есть;
- какие контроллеры за что отвечают;
- где лежит бизнес-логика;
- где есть признаки architectural debt;
- из каких мест лучше начинать рефакторинг.

---

## 1. Общая картина

Текущая архитектура backend — это Laravel monolith с несколькими слоями:

- `routes/*` — маршруты;
- `app/Http/Controllers/Api/V1/*` — пользовательский API;
- `app/Http/Controllers/Admin/*` — Blade-admin;
- `app/Services/*` — частично вынесенная бизнес-логика;
- `app/Models/*` — Eloquent модели;
- `resources/views/admin/*` — admin UI;
- jobs/events/channels — realtime и import infrastructure.

### Главный архитектурный вывод

Бизнес-логика распределена неравномерно:

- часть в контроллерах;
- часть в сервисах;
- часть в моделях;
- часть в middleware;
- часть прямо в запросах к Eloquent/DB.

Именно поэтому нужен не «переписон всего», а постепенное выравнивание структуры.

Этот документ описывает фактическое переходное состояние, а не идеальную clean/SOLID architecture. Формулировки про будущие actions/query/services ниже являются рекомендациями для refactor waves, а не уже существующей структурой.

---

## 2. Public catalog module

### Routes
- `/api/v1/public/anime*`
- `/api/v1/public/tags`
- `/api/v1/public/episodes*`
- `/api/v1/public/schedule`

### Controllers
- `Api\V1\AnimeController`
- `Api\V1\EpisodeController`
- `Api\V1\TagController`
- `Api\V1\ScheduleController`

### Related models
- `Anime`
- `Episode`
- `Tag`

### Related services
- `BannerService`
- внешние integration services существуют в `app/Services`, но не все они являются частью catalog query flow

### Текущая ответственность

#### `AnimeController`
Отвечает за:
- каталог anime;
- фильтрацию;
- поиск;
- сортировку;
- детальную страницу anime;
- community stats;
- banner response;
- рекомендации.

#### Architectural note
`AnimeController` сейчас слишком тяжелый:
- query construction;
- fallback logic;
- внешний HTTP вызов;
- raw SQL similarity logic;
- response composition.

### Вывод
Это один из первых кандидатов на вынос в:
- query classes;
- recommendation service/query;
- action classes.

---

## 3. Auth module

### Routes
- `/api/v1/auth/register`
- `/api/v1/auth/login`
- `/api/v1/auth/logout`
- `/api/v1/user`

### Controller
- `Api\V1\AuthController`

### Related models
- `User`
- `Role`

### Текущая ответственность
- регистрация;
- логин;
- me;
- logout;
- выдача токена;
- частично user initialization.

### Architectural note
Критичный модуль. Любой рефакторинг делать только после baseline и тестов.

---

## 4. Profile and statistics module

### Routes
- `/api/v1/profile/me`
- `/api/v1/profile/me/avatar`
- `/api/v1/statistics/me`
- `/api/v1/statistics/me/episodes-summary`
- `/api/v1/public/users/{userId}/statistics`

### Controllers
- `Api\V1\UserProfileController`
- `Api\V1\UserStatisticsController`

### Services
- `UserProfileService`
- `UserStatisticsService`

### Related models
- `User`
- `WatchHistory`
- `Rating`
- `Comment`
- `Anime`

### Текущая ответственность
- данные профиля;
- update profile;
- avatar upload;
- user statistics;
- watched summary;
- profile aggregates.

### Architectural note
Это один из модулей, где service layer уже есть, но логика частично дублируется.

### Риск
- дублирование агрегаций;
- смешение query/business/presentation responsibility.

---

## 5. Comments module

### Routes
- `/api/v1/public/anime/{anime}/comments`
- `/api/v1/comments`
- `/api/v1/my-comments`

### Controller
- `Api\V1\CommentsController`

### Related requests/resources
- `StoreCommentRequest`
- `UpdateCommentRequest`
- `CommentResource`

### Related models
- `Comment`
- `Anime`
- `User`

### Текущая ответственность
- public comments list;
- create comment;
- update own comment;
- delete own comment;
- my comments list;
- comments counter updates.

### Architectural note
Сейчас контроллер сам:
- открывает транзакции;
- правит агрегаты (`comments_count`);
- проверяет ownership;
- возвращает response.

### Вывод
Нужен вынос в:
- comment actions;
- policies;
- centralized counter strategy.

---

## 6. Ratings module

### Routes
- `/api/v1/ratings`
- `/api/v1/ratings/anime/{animeId}`

### Controller
- `Api\V1\RatingsController`

### Related request/resource
- `StoreRatingRequest`
- `RatingResource`

### Related models
- `Rating`
- `Anime`

### Текущая ответственность
- list my ratings;
- upsert rating;
- delete rating;
- read my rating for anime;
- recalculate anime rating aggregate.

### Architectural note
Рейтинг пересчитывается вручную прямо в контроллере.

### Вывод
Нужен:
- `StoreRatingAction`;
- `DeleteRatingAction`;
- отдельный aggregate recalculation service/action.

---

## 7. Favorites module

### Routes
- `/api/v1/favorites*`

### Controller
- `Api\V1\FavoritesController`

### Related models
- `Favorite`
- `Anime`
- `User`

### Текущая ответственность
- список избранного;
- добавление;
- удаление;
- проверка наличия.

### Architectural note
Относительно компактный модуль, вероятно рефакторится позже после heavy modules.

---

## 8. Watch history module

### Routes
- `/api/v1/watch-history*`

### Controller
- `Api\V1\WatchHistoryController`

### Related requests
- `UpdateWatchHistoryRequest`

### Related models
- `WatchHistory`
- `Episode`
- `Anime`

### Текущая ответственность
- список истории;
- upsert прогресса;
- show entry;
- delete entry;
- anime-specific history;
- last watched episode.

### Architectural note
Контроллер отвечает и за прогресс, и за агрегацию, и за timeline, и за response formatting.

### Вывод
Нужен вынос в:
- `SyncWatchHistoryAction`;
- `WatchHistoryQuery`;
- единые DTO для progress payload.

---

## 9. User anime list module

### Routes
- `/api/v1/anime/{anime}/status`
- `/api/v1/anime/{anime}/user-status`
- `/api/v1/anime/{anime}/episodes-watched/{episodesWatched}`
- `/api/v1/my-anime-list/{status?}`

### Controller
- `Api\V1\UserAnimeListController`

### Related request
- `UpdateAnimeStatusRequest`

### Related models
- `User`
- `Anime`
- pivot `anime_user`

### Текущая ответственность
- list user anime;
- get status for anime;
- update anime status;
- detach from list;
- update episodes watched.

### Architectural note
Бизнес-правила pivot-состояния живут прямо в контроллере.

### Вывод
Нужны:
- list query;
- status action;
- episodes watched action;
- validation tightening.

---

## 10. Friendship module

### Routes
- `/api/v1/friends*`
- `/api/v1/users/search`

### Controller
- `Api\V1\FriendshipController`

### Related models
- `Friendship`
- `User`

### Текущая ответственность
- список друзей;
- входящие/исходящие заявки;
- count badge;
- send request;
- accept/decline;
- status;
- users search;
- user formatting.

### Architectural note
Контроллер содержит почти весь use case flow.

### Вывод
Это heavy social module, подходящий для early refactor wave.

---

## 11. Watch party module

### Routes
- `/api/v1/watch-party*`
- broadcast channels `watch-party.{code}`
- user invite channel `user.{userId}`

### Controller
- `Api\V1\WatchPartyController`

### Related models
- `WatchPartyRoom`
- `WatchPartyParticipant`
- `WatchPartyMessage`
- `Anime`
- `Episode`

### Related events
- `FriendInviteEvent`
- `ChatMessageEvent`
- `PlayerSyncEvent`
- `RoomClosedEvent`

### Текущая ответственность
- room creation;
- join/leave;
- capacity checks;
- host checks;
- sync state;
- chat history;
- send message;
- invite friend;
- close room;
- room formatting.

### Architectural note
Один из самых нагруженных контроллеров по предметной логике.

### Вывод
Это **топ-1 кандидат** на декомпозицию в actions/use cases.

---

## 12. Payment module

### Routes
- `/api/v1/payment/premium`

### Controller
- `PaymentController`

### Note
Изолированный модуль. Пока не трогать без причины.

---

## 13. Admin dashboard module

### Routes
- `/admin/dashboard`

### Controller
- `Admin\DashboardController`

### Related models
- `Anime`
- `Episode`
- `Tag`
- `User`
- `ImportLog`

### Responsibility
- собирает dashboard metrics для Blade-admin.

### Migration note
Нужно сделать admin dashboard API.

---

## 14. Admin anime management module

### Routes
- `/admin/anime*`

### Controller
- `Admin\AnimeManagementController`

### Related models
- `Anime`
- `Tag`
- `Episode`
- `AuditLog`
- `BlacklistedAnime`

### Responsibility
- anime CRUD;
- search/filter;
- tags sync;
- blacklist on delete;
- bulk delete;
- audit logging.

### Architectural note
Смешаны:
- CRUD;
- blacklist policy;
- audit logging;
- view rendering.

### Migration note
Должен стать admin API + Next UI.

---

## 15. Admin episodes management module

### Routes
- `/admin/episodes*`

### Controller
- `Admin\EpisodeManagementController`

### Related models/jobs
- `Episode`
- `Anime`
- `EpisodesImportJob`
- `AuditLog`

### Responsibility
- episode list/edit/delete;
- import for one anime;
- import all;
- bulk import.

### Architectural note
`importAll()` использует `Anime::all()`, что не масштабируется хорошо.

### Migration note
Один из первых admin API кандидатов.

---

## 16. Admin users management module

### Routes
- `/admin/users*`

### Controller
- `Admin\UserManagementController`

### Related models
- `User`
- `AuditLog`

### Responsibility
- list/search/show;
- ban/unban;
- delete.

### Migration note
Подходит для admin API второй волны.

---

## 17. Admin tags management module

### Routes
- `/admin/tags*`

### Controller
- `Admin\TagManagementController`

### Related models
- `Tag`
- `AuditLog`

### Responsibility
- tags CRUD;
- search;
- audit logging.

---

## 18. Admin comment moderation module

### Routes
- `/admin/comments*`

### Controller
- `Admin\CommentModerationController`

### Related models
- `Comment`
- `Anime`

### Responsibility
- moderation list;
- approve/reject;
- delete.

### Architectural note
Есть рискованная ручная работа с `comments_count`.

---

## 19. Admin import management module

### Routes
- `/admin/import*`

### Controller
- `Admin\ImportManagementController`

### Related models/jobs
- `ImportLog`
- `ImportAnimeJob`
- `AuditLog`

### Responsibility
- import dashboard;
- run import;
- logs list.

### Migration note
Очень важен для demo и операционной части проекта.

---

## 20. Admin audit log module

### Routes
- `/admin/audit-logs`

### Controller
- `Admin\AuditLogController`

### Related models
- `AuditLog`

### Responsibility
- просмотр audit trails;
- фильтрация логов.

---

## 21. Infrastructure and cross-cutting modules

### Channels
- `routes/channels.php`
- realtime access rules for watch party / user invites

### Middleware
- security-related middleware
- admin middleware
- auth boundaries

### Services
- import services
- profile/statistics services
- banner/external integrations
- внешние integration services: `AnilibriaService`, `KodikService`, `ShikimoriImportService`, `VideoCdnService`, `AiDescriptionService`

### Note
Именно здесь много cross-cutting technical debt:
- security policy;
- input sanitization;
- operational config;
- external API coupling.

---

## 22. Где больше всего технического долга

## High debt
- `Api\V1\AnimeController`
- `Api\V1\WatchPartyController`
- `Api\V1\FriendshipController`
- `Api\V1\CommentsController`
- `Api\V1\RatingsController`
- `Admin\AnimeManagementController`
- `Admin\EpisodeManagementController`
- security middleware

## Medium debt
- `UserAnimeListController`
- `WatchHistoryController`
- `UserProfileService`
- `UserStatisticsService`
- `CommentModerationController`
- import orchestration

## Lower priority
- favorites module;
- tag CRUD;
- audit logs list.

---

## 23. Откуда начинать рефакторинг

### Первая волна
1. `AnimeController`
2. `WatchPartyController`
3. `FriendshipController`

### Вторая волна
4. `CommentsController`
5. `RatingsController`
6. `WatchHistoryController`
7. `UserAnimeListController`

### Параллельная линия
8. Admin API extraction из:
- dashboard
- anime management
- episodes management
- import management

---

## 24. Итог

Этот responsibility map нужен как основа для любого структурного рефакторинга. Перед переносом логики стоит отвечать на 3 вопроса:

1. Это user API модуль, social module или admin module?
2. Где сейчас реально живет бизнес-логика?
3. Во что это должно превратиться: action, query, policy, service или admin API controller?
