# Architecture cleanup report — 2026-05-08

Этот документ фиксирует изменения, сделанные в рамках аудита и первичной чистки frontend/backend связки.

## Правило на будущее

Все последующие крупные изменения по проекту должны сопровождаться отдельным документом в `docs/` с описанием:

- цели изменения;
- затронутых файлов;
- причин выбранного решения;
- выполненных проверок;
- оставшихся рисков/следующих шагов.

## Frontend

### API proxy

Затронутые файлы:

- `aniyume/app/api/external/[...path]/route.ts`
- `aniyume/next.config.ts`

Что сделано:

- исправлена проблема двойного `/public/public/...` при запросах к публичным backend endpoint'ам;
- proxy теперь нормализует публичные пути `anime`, `episodes`, `tags`, `schedule`;
- конфликтующий rewrite `/api/external/:path*` удален из `next.config.ts`;
- `/api-storage/:path*` rewrite оставлен.

### P2P player / SSR

Затронутый файл:

- `aniyume/components/watch/AnimePlayer.tsx`

Что сделано:

- `P2PVideoPlayer` переведен на dynamic import с `ssr: false`;
- исправлено падение `next build` из-за `node-datachannel`/`p2p-media-loader` в Turbopack SSR pipeline.

### Watch history

Затронутые файлы:

- `aniyume/app/anime/[id]/page.tsx`
- `aniyume/hooks/useWatchTracker.ts`

Что сделано:

- добавлено обязательное backend-поле `delta_time`;
- `navigator.sendBeacon` заменен на `fetch(..., { keepalive: true })`, чтобы передавать `Authorization: Bearer ...`.

### Bookmarks / anime list

Затронутый файл:

- `aniyume/app/(navigation)/bookmarks/page.tsx`

Что сделано:

- удаление из пользовательских списков больше не вызывает несуществующий `DELETE /my-anime-list/{id}`;
- теперь используется `POST /anime/{id}/status` с `status: not_watching`.

### Schedule page

Затронутый файл:

- `aniyume/app/(navigation)/schedule/page.tsx`

Что сделано:

- frontend больше не строит расписание через `anime.id % 7`;
- подключен backend endpoint `/api/external/schedule`.

### Next image config

Затронутый файл:

- `aniyume/next.config.ts`

Что сделано:

- удален deprecated `images.domains`;
- домены перенесены в `images.remotePatterns`.

### Reverb env example

Затронутый файл:

- `aniyume/.env.example`

Что сделано:

- добавлен пример frontend env для backend URL и Reverb websocket параметров.

## Backend

### Episode import runtime errors

Затронутые файлы:

- `aniyume-backend/app/Jobs/EpisodesImportJob.php`
- `aniyume-backend/app/Http/Controllers/Admin/EpisodeManagementController.php`
- `aniyume-backend/routes/api.php`

Что сделано:

- добавлен отсутствующий `EpisodesImportJob`;
- добавлен отсутствующий метод `bulkImport()`;
- исправлен route watched episodes на `/watched/{episodesWatched}`.

### Anime API pagination and statuses

Затронутый файл:

- `aniyume-backend/app/Http/Controllers/Api/V1/AnimeController.php`

Что сделано:

- добавлена поддержка `per_page` с лимитом 1–100;
- добавлены aliases статусов `releasing -> ongoing`, `upcoming/anons -> planned`;
- удалены legacy methods: `episodes`, `episode`, `search`, `updateStatus`, `getUserStatus`.

### Admin security

Затронутый файл:

- `aniyume-backend/routes/web.php`

Что сделано:

- admin routes теперь используют middleware `auth` + `admin`.

### Laravel 12 middleware/schedule

Затронутые файлы:

- `aniyume-backend/bootstrap/app.php`
- `aniyume-backend/routes/console.php`
- `aniyume-backend/app/Http/Kernel.php`
- `aniyume-backend/app/Console/Kernel.php`

Что сделано:

- API middleware stack приведен к Laravel 12 bootstrap config;
- добавлен Sanctum stateful middleware в `bootstrap/app.php`;
- schedule `episodes:sync-ongoing --days=1` перенесен в `routes/console.php`;
- legacy Kernel-файлы помечены как совместимость, не основной runtime config;
- из legacy HTTP Kernel убраны ссылки на несуществующие local middleware aliases.

### Watch Party broadcast

Затронутые файлы:

- `aniyume-backend/app/Events/ChatMessageEvent.php`
- `aniyume-backend/app/Events/RoomClosedEvent.php`
- `aniyume-backend/app/Events/FriendInviteEvent.php`

Что сделано:

- события переведены на `ShouldBroadcastNow`, чтобы realtime не зависел от queue worker;
- live chat event теперь возвращает `type: message`.

### Tests

Затронутый файл:

- `aniyume-backend/tests/Feature/ExampleTest.php`

Что сделано:

- тест `/` теперь ожидает redirect на `/admin/login`, что соответствует фактическому поведению приложения.

### Routes cleanup

Затронутый файл:

- `aniyume-backend/routes/api.php`

Что сделано:

- удален дублирующий блок `/api/v1/anime-list/...`, так как frontend и актуальный API используют `/my-anime-list/...` и `/anime/{id}/status`.

### Controller cleanup

Затронутые файлы:

- `aniyume-backend/app/Http/Controllers/Api/V1/UserProfileController.php`
- `aniyume-backend/app/Http/Controllers/Api/V1/TagController.php`

Что сделано:

- удалены неиспользуемые `show()` methods и связанные imports.

### Import commands

Затронутые файлы:

- `aniyume-backend/app/Console/Commands/ImportEpisodes.php`
- `aniyume-backend/app/Console/Commands/ResetAnime.php`

Что сделано:

- `import:episodes` обозначен как каноническая команда;
- `episodes:import` оставлен как backward-compatible alias;
- подсказка в `ResetAnime` обновлена на `php artisan import:episodes --limit=100`.

## Проверки

Выполнены проверки:

- `npm run build` в `aniyume` — успешно;
- `php artisan test` в `aniyume-backend` — успешно, 2 теста проходят;
- `php artisan schedule:list` — показывает `import:anime`, `import:episodes --limit=200`, `episodes:sync-ongoing --days=1`;
- точечные `php -l` для измененных PHP-файлов — без syntax errors.

## Известные оставшиеся задачи

- проверить и очистить временные root-файлы backend;
- проверить неиспользуемые сервисы (`AnilistImportService` и др.);
- проверить модели `Studio`, `Genre`, `UserVideo`, `UserCollection`;
- решить судьбу временной документации/дампов в backend root;
- проверить production Reverb env и reverse proxy;
- при необходимости перевести broadcast auth на единый endpoint без дублирования.
