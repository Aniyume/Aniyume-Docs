# Admin Next migration plan

## Назначение

Документ описывает минимальный набор экранов будущего Next.js admin client и migration waves от текущей Blade-admin панели к JSON API + отдельному admin frontend.

Связанные документы:

- `docs/admin-api-migration-map.md`
- `docs/admin-api-design-spec.md`

## Целевой frontend layout

Минимальный Next admin должен иметь:

- отдельный `/login` без основного layout;
- protected app layout для всех admin pages;
- sidebar navigation;
- topbar с текущим admin user и logout;
- общий API client для `/api/v1/admin/*`;
- global handling `401`/`403`;
- reusable table, pagination, filters, form validation errors, confirm dialogs, toast notifications.

## Минимальный набор экранов

### P0 screens — нужны для MVP parity

#### `/login`

Назначение:

- admin login;
- показ ошибок `401`, `403`, `422`;
- redirect на `/dashboard` после успеха.

API:

- `POST /api/v1/admin/auth/login`
- `GET /api/v1/admin/auth/me` после инициализации protected layout.

#### Protected layout

Назначение:

- проверить auth state;
- загрузить current admin;
- logout;
- role guard.

API:

- `GET /api/v1/admin/auth/me`
- `POST /api/v1/admin/auth/logout`

#### `/dashboard`

Назначение:

- summary cards: anime, episodes, tags, users;
- latest anime;
- recent imports;
- charts/tables for anime by status/type.

API:

- `GET /api/v1/admin/dashboard`

#### `/anime`

Назначение:

- table/list anime;
- search/filter by status/type/tag/nsfw;
- pagination/sort;
- bulk delete;
- links to create/show/edit.

API:

- `GET /api/v1/admin/anime`
- `POST /api/v1/admin/anime/bulk-delete`
- `DELETE /api/v1/admin/anime/{id}`

#### `/anime/create`

Назначение:

- create anime form;
- tag multi-select;
- backend validation errors.

API:

- `POST /api/v1/admin/anime`
- `GET /api/v1/admin/tags?per_page=100&search=...` for selector.

#### `/anime/[id]`

Назначение:

- anime detail;
- related episodes list;
- delete/import actions.

API:

- `GET /api/v1/admin/anime/{id}`
- `DELETE /api/v1/admin/anime/{id}`
- `POST /api/v1/admin/episodes/{animeId}/import`

#### `/anime/[id]/edit`

Назначение:

- edit anime form;
- sync/detach tags.

API:

- `GET /api/v1/admin/anime/{id}`
- `PUT /api/v1/admin/anime/{id}`
- `GET /api/v1/admin/tags?per_page=100&search=...`

#### `/episodes`

Назначение:

- table episodes;
- filter by anime/search;
- import all/bulk import controls;
- delete episode.

API:

- `GET /api/v1/admin/episodes`
- `POST /api/v1/admin/episodes/import-all`
- `POST /api/v1/admin/episodes/bulk-import`
- `DELETE /api/v1/admin/episodes/{id}`

#### `/episodes/[id]/edit`

Назначение:

- edit episode metadata/player fields.

API:

- `GET /api/v1/admin/episodes/{id}`
- `PUT /api/v1/admin/episodes/{id}`

#### `/imports`

Назначение:

- latest import status;
- import counters;
- run initial/update import.

API:

- `GET /api/v1/admin/imports/dashboard`
- `POST /api/v1/admin/imports/run`

#### `/imports/logs`

Назначение:

- paginated import logs;
- status/type filters.

API:

- `GET /api/v1/admin/imports/logs`

### P1 screens — soon after MVP

#### `/users`

Назначение:

- users table;
- search/filter banned;
- ban/unban/delete actions.

API:

- `GET /api/v1/admin/users`
- `POST /api/v1/admin/users/{id}/ban`
- `POST /api/v1/admin/users/{id}/unban`
- `DELETE /api/v1/admin/users/{id}`

#### `/users/[id]`

Назначение:

- user profile details;
- ban/unban/delete controls;
- activity summary.

API:

- `GET /api/v1/admin/users/{id}`

#### `/tags`

Назначение:

- tags table;
- search;
- create/edit/delete.

API:

- `GET /api/v1/admin/tags`
- `POST /api/v1/admin/tags`
- `PUT /api/v1/admin/tags/{id}`
- `DELETE /api/v1/admin/tags/{id}`

#### `/tags/create` and `/tags/[id]/edit`

Назначение:

- dedicated tag forms if inline table editing is not enough.

API:

- `GET /api/v1/admin/tags/{id}`
- `POST /api/v1/admin/tags`
- `PUT /api/v1/admin/tags/{id}`

#### `/comments`

Назначение:

- comment moderation list;
- approve/reject/delete.

API:

- `GET /api/v1/admin/comments`
- `POST /api/v1/admin/comments/{id}/approve`
- `POST /api/v1/admin/comments/{id}/reject`
- `DELETE /api/v1/admin/comments/{id}`

### P2 screens — can be later

#### `/audit-logs`

Назначение:

- audit log table;
- action/user/date filters.

API:

- `GET /api/v1/admin/audit-logs`

#### Future analytics/settings

Optional after parity:

- deeper import metrics;
- moderation analytics;
- admin settings;
- more granular permissions.

## Migration waves

### Wave 0 — design freeze and parity checklist

Цель: зафиксировать contract до кодинга.

Что сделать:

- approve `admin-api-design-spec.md`;
- отметить все Blade use cases в checklist;
- выбрать auth mode: Bearer token or Sanctum cookie SPA;
- согласовать CORS/cookie/token storage policy for Next admin;
- определить, где будет жить `aniyume-admin`.

Что нельзя делать:

- удалять Blade routes/views/controllers;
- менять public API endpoints;
- внедрять breaking DB changes без отдельной миграционной задачи.

### Wave 1 — API foundation and P0 endpoints

Цель: дать Next admin MVP backend surface.

Сначала реализовать:

1. admin auth API: login/logout/me;
2. shared admin middleware/guard for JSON;
3. dashboard API;
4. anime API including bulk delete and blacklist behavior;
5. episodes API including import endpoints;
6. imports dashboard/logs/run API.

Почему сначала это:

- без auth невозможно безопасно тестировать остальные endpoints;
- dashboard/anime/episodes/imports покрывают core operational admin сценарии;
- эти области имеют P0 приоритет в текущем migration map.

### Wave 2 — Next admin MVP

Цель: первый рабочий отдельный admin UI без удаления Blade.

Сделать:

- Next app skeleton;
- login/protected layout;
- dashboard;
- anime list/create/detail/edit;
- episodes list/edit/import controls;
- imports dashboard/logs.

Acceptance:

- admin can login/logout without Blade;
- admin can perform P0 CRUD/import tasks through Next UI;
- JSON validation errors are shown in forms;
- `401`/`403` handled globally.

### Wave 3 — P1 parity

Цель: покрыть use cases, которые нужны сразу после MVP.

Сделать:

- users API/UI;
- tags API/UI;
- comments moderation API/UI;
- missing audit logs for P1 mutations where needed.

Acceptance:

- user search/profile/ban/unban/delete available;
- tag CRUD available;
- comment approve/reject/delete available.

### Wave 4 — P2 observability and hardening

Цель: эксплуатационная прозрачность и безопасность.

Сделать:

- audit logs API/UI;
- action filters/date filters;
- rate limiting for login/import mutation endpoints;
- confirmation UX for destructive actions;
- optional admin permission granularity beyond single `admin` role.

### Wave 5 — parity verification and Blade retirement

Цель: удалить legacy layer только после доказанной parity.

До удаления нужно проверить:

- all P0/P1 scenarios work in Next admin;
- API feature tests pass;
- manual regression checklist complete;
- no team/process still depends on `/admin/*` Blade pages;
- rollback plan exists.

Only after that can be removed:

- `routes/web.php` admin route group;
- `app/Http/Controllers/Admin/*` legacy Blade controllers;
- `resources/views/admin/*`;
- `resources/views/layouts/admin.blade.php` if only used by admin.

## Что можно оставить на потом

- audit logs UI if not required for MVP operations;
- advanced analytics;
- granular permissions/roles beyond `admin`;
- advanced bulk UX and background job progress streaming;
- comment `pending` tri-state if DB currently has only boolean `is_approved`;
- full admin settings section.

## Что нельзя удалять до migration parity

- Blade admin routes under `/admin/*`;
- existing Blade admin views;
- existing `App\Http\Controllers\Admin\*` controllers;
- current auth/session behavior used by Blade;
- blacklist behavior on anime deletion;
- import job dispatch behavior;
- existing public `/api/v1/public/*` and user API endpoints.

## Main risks

- Auth mode ambiguity: Bearer token is simpler for API, cookie/Sanctum SPA is often safer for browser admin; choose before implementation.
- Current comment moderation appears boolean-only; a `pending` state needs DB support or explicit exclusion.
- Some current destructive actions have limited safeguards; Next UI should add confirmations, but backend should also validate dangerous operations.
- Large anime/tag selectors should not preload entire catalogs into every page.
- Removing Blade before P0/P1 parity would break existing admin operations.
