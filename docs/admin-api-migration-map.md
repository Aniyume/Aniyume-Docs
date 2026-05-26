# Admin API migration map

## Назначение

Этот документ описывает, как встроенная Blade-admin панель Laravel должна быть перенесена в модель:

- **Laravel backend API-only**
- **отдельный Next.js admin client**

Цели:

- зафиксировать текущий admin функционал;
- определить admin API endpoints, которые нужно создать;
- не потерять ни один admin use case при миграции;
- понять, какие Blade-экраны должны быть заменены в первую очередь.

---

## 1. Текущее состояние

Сейчас admin реализован через:

- `routes/web.php`
- `app/Http/Controllers/Admin/*`
- `resources/views/admin/*`

Модель работы:

- login через web form;
- session/auth middleware;
- Blade templates;
- direct Eloquent + view rendering.

Важно: это описание фиксирует **current state**, а не уже реализованную целевую архитектуру. На момент baseline admin API под `/api/v1/admin/*` отсутствует; все перечисленные ниже admin API endpoints являются planned contract для миграции.

### Проблемы текущего подхода

- backend совмещает API и web-admin в одном слое;
- логика admin не переиспользуется фронтендом как API;
- сильная связность backend и Blade;
- сложнее сопровождать и тестировать;
- архитектура хуже смотрится на защите диплома.

---

## 2. Целевая модель

Нужно перейти к структуре:

- `aniyume-backend` — только API и инфраструктура;
- `aniyume-admin` — отдельный Next.js admin frontend;
- admin взаимодействует с backend по JSON API.

### Целевой auth flow

Рекомендуемый вариант:

- admin login через backend API;
- backend проверяет роль admin;
- admin frontend хранит auth state безопасным способом;
- backend использует Sanctum/token или cookie-based admin auth contract;
- доступ к admin API только для пользователей с admin role.

---

## 3. Карта текущего admin функционала

## 3.1. Admin authentication

### Текущее состояние

**Routes:**
- `GET /admin/login`
- `POST /admin/login`
- `GET /admin/logout`
- `POST /admin/logout`

**Controller:**
- `Admin\AuthController`

**Blade views:**
- `resources/views/admin/auth/*`

### Что нужно в новой архитектуре

**Admin API:**
- `POST /api/v1/admin/auth/login`
- `POST /api/v1/admin/auth/logout`
- `GET /api/v1/admin/auth/me`

**Next admin pages:**
- `/login`
- protected layout/guard

### Priority
**P0**

---

## 3.2. Admin dashboard

### Текущее состояние

**Route:**
- `GET /admin/dashboard`

**Controller:**
- `DashboardController@index`

**Что показывает:**
- total anime;
- total episodes;
- total tags;
- total users;
- recent imports;
- anime by status;
- anime by type;
- latest anime.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/dashboard`

**Ответ должен содержать:**
- summary metrics;
- latest anime;
- recent imports;
- chart-ready aggregated data.

**Next admin page:**
- `/dashboard`

### Priority
**P0**

---

## 3.3. User management

### Текущее состояние

**Routes:**
- `GET /admin/users`
- `GET /admin/users/create`
- `POST /admin/users`
- `GET /admin/users/{user}`
- `GET /admin/users/{user}/edit`
- `PUT/PATCH /admin/users/{user}`
- `POST /admin/users/{user}/ban`
- `POST /admin/users/{user}/unban`
- `DELETE /admin/users/{user}`

**Controller:**
- `UserManagementController`

**Текущие use cases:**
- список пользователей;
- поиск;
- просмотр профиля;
- resource routes create/store/edit/update зарегистрированы Laravel route resource и требуют отдельной проверки фактического UI/готовности;
- бан/разбан;
- удаление.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/users`
- `GET /api/v1/admin/users/{id}`
- `POST /api/v1/admin/users/{id}/ban`
- `POST /api/v1/admin/users/{id}/unban`
- `DELETE /api/v1/admin/users/{id}`

**Дополнительно желательно:**
- pagination meta;
- filters;
- sort;
- audit trail in response if needed.

**Next admin pages:**
- `/users`
- `/users/[id]`

### Priority
**P1**

---

## 3.4. Anime management

### Текущее состояние

**Routes:**
- `GET /admin/anime`
- `GET /admin/anime/create`
- `POST /admin/anime`
- `GET /admin/anime/{anime}`
- `GET /admin/anime/{anime}/edit`
- `PUT/PATCH /admin/anime/{anime}`
- `DELETE /admin/anime/{anime}`
- `POST /admin/anime/bulk-delete`

**Controller:**
- `AnimeManagementController`

**Текущие use cases:**
- list/search/filter;
- create;
- update;
- delete;
- bulk delete;
- tags sync;
- blacklist on delete;
- show related episodes.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/anime`
- `POST /api/v1/admin/anime`
- `GET /api/v1/admin/anime/{id}`
- `PUT /api/v1/admin/anime/{id}`
- `DELETE /api/v1/admin/anime/{id}`
- `POST /api/v1/admin/anime/bulk-delete`

**Дополнительно желательно:**
- отдельный endpoint для tags attach/sync при необходимости;
- формализованный response schema;
- validation errors JSON-friendly.

**Next admin pages:**
- `/anime`
- `/anime/create`
- `/anime/[id]`
- `/anime/[id]/edit`

### Priority
**P0**

---

## 3.5. Tag management

### Текущее состояние

**Routes:**
- `GET /admin/tags`
- `GET /admin/tags/create`
- `POST /admin/tags`
- `GET /admin/tags/{tag}/edit`
- `PUT/PATCH /admin/tags/{tag}`
- `DELETE /admin/tags/{tag}`

**Controller:**
- `TagManagementController`

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/tags`
- `POST /api/v1/admin/tags`
- `GET /api/v1/admin/tags/{id}`
- `PUT /api/v1/admin/tags/{id}`
- `DELETE /api/v1/admin/tags/{id}`

**Next admin pages:**
- `/tags`
- `/tags/create`
- `/tags/[id]/edit`

### Priority
**P1**

---

## 3.6. Episode management and imports

### Текущее состояние

**Routes:**
- `GET /admin/episodes`
- `GET /admin/episodes/{episode}/edit`
- `PUT /admin/episodes/{episode}`
- `POST /admin/episodes/import-all`
- `POST /admin/episodes/{anime}/import`
- `POST /admin/episodes/bulk-import`
- `DELETE /admin/episodes/{id}`

**Controller:**
- `EpisodeManagementController`

**Use cases:**
- список эпизодов;
- фильтр по anime;
- редактирование episode fields;
- импорт по одному anime;
- массовый импорт;
- удаление episode.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/episodes`
- `GET /api/v1/admin/episodes/{id}`
- `PUT /api/v1/admin/episodes/{id}`
- `DELETE /api/v1/admin/episodes/{id}`
- `POST /api/v1/admin/episodes/import-all`
- `POST /api/v1/admin/episodes/{animeId}/import`
- `POST /api/v1/admin/episodes/bulk-import`

**Next admin pages:**
- `/episodes`
- `/episodes/[id]/edit`
- import controls embedded in anime/episodes pages

### Priority
**P0**

---

## 3.7. Comment moderation

### Текущее состояние

**Routes:**
- `GET /admin/comments`
- `POST /admin/comments/{comment}/approve`
- `POST /admin/comments/{comment}/reject`
- `DELETE /admin/comments/{comment}`

**Controller:**
- `CommentModerationController`

**Use cases:**
- список комментариев;
- фильтр по approved/rejected;
- approve/reject;
- delete.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/comments`
- `POST /api/v1/admin/comments/{id}/approve`
- `POST /api/v1/admin/comments/{id}/reject`
- `DELETE /api/v1/admin/comments/{id}`

**Next admin pages:**
- `/comments`

### Priority
**P1**

---

## 3.8. Import management

### Текущее состояние

**Routes:**
- `GET /admin/import`
- `POST /admin/import/run`
- `GET /admin/import/logs`

**Controller:**
- `ImportManagementController`

**Use cases:**
- запуск initial/update import;
- просмотр latest import и агрегатов;
- просмотр import logs.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/imports/dashboard`
- `POST /api/v1/admin/imports/run`
- `GET /api/v1/admin/imports/logs`

**Next admin pages:**
- `/imports`
- `/imports/logs`

### Priority
**P0**

---

## 3.9. Audit logs

### Текущее состояние

**Route:**
- `GET /admin/audit-logs`

**Controller:**
- `AuditLogController`

**Use cases:**
- список логов;
- фильтр по action;
- фильтр по user;
- фильтр по дате.

### Что нужно в новой архитектуре

**Admin API:**
- `GET /api/v1/admin/audit-logs`

**Next admin pages:**
- `/audit-logs`

### Priority
**P2**

---

## 4. Предлагаемая структура admin API

Ни один endpoint из этого раздела не считается существующим, пока он не появится в `routes/api.php` и не будет подтвержден `php artisan route:list`/тестами. Это migration target, а не текущий API surface.

### Вариант структуры роутов

```text
/api/v1/admin/auth/*
/api/v1/admin/dashboard
/api/v1/admin/users/*
/api/v1/admin/anime/*
/api/v1/admin/tags/*
/api/v1/admin/episodes/*
/api/v1/admin/comments/*
/api/v1/admin/imports/*
/api/v1/admin/audit-logs
```

### Рекомендуемые контроллеры

- `App\Http\Controllers\Api\Admin\AuthController`
- `DashboardController`
- `UsersController`
- `AnimeController`
- `TagsController`
- `EpisodesController`
- `CommentsController`
- `ImportsController`
- `AuditLogsController`

---

## 5. Приоритет миграции

## Wave 1 — обязательно до отказа от Blade
- admin auth API
- dashboard API
- anime API
- episodes API
- imports API

## Wave 2 — сразу после первой волны
- users API
- tags API
- comments moderation API

## Wave 3 — можно позже
- audit logs improvements
- analytics enhancements
- bulk UX enhancements

---

## 6. Что нельзя удалять сразу

До завершения migration parity нельзя удалять:

- `routes/web.php` admin routes;
- `resources/views/admin/*`;
- `app/Http/Controllers/Admin/*`.

Сначала нужно:

1. создать API-эквиваленты;
2. поднять Next admin MVP;
3. проверить сценарии;
4. только потом удалять legacy Blade layer.

---

## 7. Definition of done для migration

Миграция admin считается завершенной, если:

- все ключевые admin use cases доступны через JSON API;
- существует отдельный `aniyume-admin` на Next.js;
- login/logout/admin guard работают без Blade;
- CRUD/import/moderation работают через новый UI;
- `routes/web.php` больше не нужен для полноценной admin работы;
- Blade-admin может быть удален без потери функциональности.

---

## 8. Итог

Этот migration map нужен как контрольный документ против потери функционала. Во время выноса admin каждый старый Blade use case должен получить:

- либо прямой API-эквивалент;
- либо улучшенный API-first вариант;
- либо осознанное исключение, если фича больше не нужна.

---

## 9. Design/spec package для следующей фазы

Для кодинга admin API следующему агенту нужно использовать этот документ как карту parity, а детальные contracts вынесены в отдельные design docs:

- `docs/admin-api-design-spec.md` — endpoint-by-endpoint JSON API contract: method, path, auth requirements, request payload, response schema, validation rules, pagination/filter/sort.
- `docs/admin-next-migration-plan.md` — минимальные экраны будущего Next admin, migration waves, что можно отложить и что нельзя удалять до завершения parity.

На текущей фазе **ничего не внедрялось в PHP-код**. Legacy Blade admin должен оставаться рабочим до завершения migration parity.
