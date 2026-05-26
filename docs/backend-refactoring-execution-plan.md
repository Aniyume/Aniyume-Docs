# Backend refactoring execution plan

## Назначение

Это **пошаговый execution plan** для приведения `aniyume-backend` к:

- более чистой архитектуре;
- лучшей производительности;
- более безопасной конфигурации;
- Docker-ready состоянию;
- готовности к выносу Blade-admin в отдельный Next.js admin;
- соответствию сильной дипломной работе.

Документ ориентирован не просто на идеи, а на **порядок реального выполнения**, чтобы можно было идти этап за этапом без хаотичного переписывания.

Связанный стратегический документ:
- `docs/backend-refactoring-thesis-master-plan.md`

---

## Главный принцип выполнения

Не делать полный переписон за один раз.

Правильный порядок:

1. зафиксировать текущее состояние;
2. закрыть критические риски;
3. стабилизировать backend ядро;
4. только потом выносить admin;
5. затем добавлять Docker/CI;
6. параллельно собирать дипломные артефакты.

---

# Phase 1. Baseline и безопасная точка старта

## Цель

Понять текущую систему до изменений и подготовить безопасный фундамент для рефакторинга.

## Что делаем

### 1.1. Зафиксировать карту backend endpoints

**Файлы:**
- `aniyume-backend/routes/api.php`
- `aniyume-backend/routes/web.php`
- `aniyume-backend/routes/channels.php`

**Задачи:**
- выписать все public API endpoints;
- выписать все protected API endpoints;
- выписать все admin web routes;
- отметить, какие из них должны остаться после выноса admin;
- отметить, какие должны быть заменены admin API.

**Результат:**
- `docs/admin-api-migration-map.md`
- `docs/backend-endpoint-inventory.md`

---

### 1.2. Зафиксировать карту модулей и ответственности

**Файлы/папки:**
- `aniyume-backend/app/Http/Controllers/Api/V1/*`
- `aniyume-backend/app/Http/Controllers/Admin/*`
- `aniyume-backend/app/Services/*`
- `aniyume-backend/app/Models/*`

**Задачи:**
- составить таблицу: модуль → контроллер → модель → сервис → маршруты;
- отметить, где бизнес-логика в контроллере;
- отметить, где есть дублирование;
- отметить, где уже можно выделять use cases.

**Результат:**
- `docs/backend-module-responsibility-map.md`

---

### 1.3. Зафиксировать baseline тестов и команд проверки

**Файлы:**
- `aniyume-backend/tests/**/*`
- `aniyume-backend/composer.json`
- `aniyume/package.json`

**Задачи:**
- зафиксировать, какие тесты уже есть;
- описать минимальный regression checklist;
- собрать команды для ручной и автоматической проверки.

**Минимальный checklist:**
- backend tests;
- frontend build;
- auth;
- public anime catalog;
- watch history;
- comments;
- ratings;
- favorites;
- watch party;
- admin login.

**Результат:**
- `docs/regression-checklist.md`

---

## Критерий завершения Phase 1

Phase завершена, если:

- есть карта endpoint'ов;
- есть карта модулей;
- есть regression checklist;
- понятно, что нельзя ломать при рефакторинге.

---

# Phase 2. Закрытие критичных технических рисков

## Цель

Убрать самые опасные проблемы до большого рефакторинга.

## Что делаем

### 2.1. Починить текущие явные ошибки и LSP-проблемы

**Проблемные файлы, замеченные при аудите:**
- `aniyume-backend/app/Http/Controllers/Api/V1/AuthController.php`
- `aniyume-backend/app/Http/Controllers/Api/V1/CommentsController.php`
- `aniyume-backend/app/Http/Controllers/Api/V1/AnimeController.php`
- `aniyume-backend/app/Http/Controllers/Api/V1/EpisodeController.php`
- `aniyume-backend/app/Http/Controllers/Admin/DashboardController.php`

**Задачи:**
- проверить реальные причины ошибок;
- устранить несовместимости сигнатур;
- убрать неправильные вызовы методов;
- исправить missing imports / facade use statements;
- прогнать `php artisan test`.

**Результат:**
- backend без базовых IDE/LSP-ошибок в ключевых файлах.

---

### 2.2. Закрыть security/config high-priority issues

**Файлы:**
- `aniyume-backend/config/cors.php`
- `aniyume-backend/config/scribe.php`
- `aniyume-backend/.env.example`
- `aniyume-backend/config/reverb.php`
- `aniyume-backend/app/Http/Middleware/SecurityShield.php`
- `aniyume-backend/app/Http/Middleware/AntiScraperMiddleware.php`

**Что делать:**
- убрать wildcard CORS для production логики;
- убрать hardcoded IP/host;
- вынести чувствительные значения в env;
- убрать dangerous input mutation из middleware;
- заменить спорную security-логику на более предсказуемую;
- нормализовать throttle/rate limit политику.

**Результат:**
- конфиг чище;
- backend безопаснее;
- меньше спорных решений на защите.

---

### 2.3. Синхронизировать README с реальным проектом

**Файл:**
- `aniyume-backend/README.md`

**Что делать:**
- убрать завышенные claims, если код им не соответствует;
- актуализировать требования к запуску;
- указать фактическую архитектуру;
- указать, что admin пока встроен, но запланирован вынос;
- отдельно указать требования к queue/reverb/import.

**Результат:**
- README становится честным и сильным техническим документом.

---

## Критерий завершения Phase 2

- исправлены явные ошибки;
- security/config high-risk issues закрыты или локализованы;
- README соответствует фактическому состоянию проекта.

---

# Phase 3. Стабилизация архитектуры backend ядра

## Цель

Подготовить backend к clean-like архитектуре без разрушения текущего функционала.

## Что делаем

### 3.1. Ввести целевую структуру слоев

**Создаваемые директории:**
- `aniyume-backend/app/Application/`
- `aniyume-backend/app/Domain/`
- `aniyume-backend/app/Infrastructure/`

**Рекомендуемая детализация:**
- `Application/Actions`
- `Application/DTOs`
- `Application/Queries`
- `Domain/Enums`
- `Domain/Contracts`
- `Domain/Policies`
- `Infrastructure/Repositories`
- `Infrastructure/External`

**Задача:**
- не переносить всё сразу;
- сначала определить структуру и использовать её для новых/refactored частей.

---

### 3.2. Вынести use cases из толстых контроллеров

## Первая волна рефакторинга

### A. Anime module
**Файл:**
- `aniyume-backend/app/Http/Controllers/Api/V1/AnimeController.php`

**Что выносить:**
- фильтрация каталога;
- сортировка;
- рекомендации;
- community stats;
- banner resolution.

**Куда:**
- `Application/Queries/AnimeCatalogQuery.php`
- `Application/Queries/AnimeRecommendationQuery.php`
- `Application/Actions/GetAnimeCommunityStatsAction.php`

---

### B. Watch Party module
**Файл:**
- `aniyume-backend/app/Http/Controllers/Api/V1/WatchPartyController.php`

**Что выносить:**
- create room;
- join room;
- leave room;
- sync player state;
- send chat message;
- invite flow;
- room close rules.

**Куда:**
- `Application/Actions/WatchParty/CreateRoomAction.php`
- `JoinRoomAction.php`
- `LeaveRoomAction.php`
- `SyncRoomStateAction.php`
- `SendRoomMessageAction.php`
- `InviteToRoomAction.php`
- `CloseRoomAction.php`

---

### C. Friendship module
**Файл:**
- `aniyume-backend/app/Http/Controllers/Api/V1/FriendshipController.php`

**Что выносить:**
- send request;
- accept/decline;
- status resolution;
- search formatting.

**Куда:**
- `Application/Actions/Friendships/SendFriendRequestAction.php`
- `AcceptFriendRequestAction.php`
- `DeclineFriendRequestAction.php`
- `Application/Queries/FriendshipStatusQuery.php`

---

### D. Comments and ratings
**Файлы:**
- `CommentsController.php`
- `RatingsController.php`

**Что выносить:**
- create/update/delete comment;
- moderation-side consistency;
- rating recalc;
- ownership/business rules.

**Куда:**
- `Application/Actions/Comments/*`
- `Application/Actions/Ratings/*`

---

### E. Watch history and anime status
**Файлы:**
- `WatchHistoryController.php`
- `UserAnimeListController.php`

**Что выносить:**
- sync progress;
- episodes watched update;
- status update;
- list query.

**Куда:**
- `Application/Actions/WatchHistory/*`
- `Application/Actions/UserAnimeList/*`
- `Application/Queries/UserAnimeListQuery.php`

---

### 3.3. Унифицировать FormRequest, Resource и Exception flow

**Файлы/папки:**
- `aniyume-backend/app/Http/Requests/**/*`
- `aniyume-backend/app/Http/Resources/**/*`
- `aniyume-backend/bootstrap/app.php`

**Что делать:**
- перевести inline validation в FormRequest там, где это критично;
- привести API resources к единому стилю;
- унифицировать формат ошибок;
- централизовать API exception responses.

---

### 3.4. Внедрить Policies/Gates системно

**Что добавить:**
- `CommentPolicy`
- `WatchPartyPolicy`
- `FriendshipPolicy`
- `AnimeManagementPolicy`
- `UserManagementPolicy`

**Цель:**
- убрать ручные проверки доступа из контроллеров.

---

## Критерий завершения Phase 3

- контроллеры заметно похудели;
- критичная бизнес-логика вынесена;
- validation/auth/response patterns стали единообразнее.

---

# Phase 4. Подготовка к выносу админки

## Цель

Подготовить backend так, чтобы admin можно было безболезненно вынести в отдельное приложение.

## Что делаем

### 4.1. Провести inventory admin-функционала

**Файлы:**
- `aniyume-backend/routes/web.php`
- `aniyume-backend/app/Http/Controllers/Admin/*`
- `aniyume-backend/resources/views/admin/*`

**Нужно описать:**
- dashboard metrics;
- users CRUD/ban/unban;
- anime CRUD/bulk delete;
- tags CRUD;
- episodes edit/import/delete;
- comments moderation;
- import runs/logs;
- audit logs.

**Результат:**
- `docs/admin-feature-inventory.md`

---

### 4.2. Создать admin API вместо Blade-only контроллеров

**Целевая папка:**
- `aniyume-backend/app/Http/Controllers/Api/Admin/`

**Первые контроллеры:**
- `DashboardController`
- `UsersController`
- `AnimeController`
- `TagsController`
- `EpisodesController`
- `CommentsController`
- `ImportsController`
- `AuditLogsController`

**Файл маршрутов:**
- либо расширение `routes/api.php`,
- либо отдельный `routes/admin-api.php` с подключением.

**Задача:**
- сначала создать API-параллель к Blade-admin;
- только потом переносить UI.

---

### 4.3. Выделить admin auth/authorization contract

**Что решить:**
- admin будет использовать тот же Sanctum/token flow или secure cookie-based flow;
- какие роли и permissions нужны;
- как admin frontend будет проверять доступ.

**Файлы для пересмотра:**
- `aniyume-backend/app/Models/User.php`
- `aniyume-backend/app/Models/Role.php`
- `aniyume-backend/database/seeders/RoleSeeder.php`
- auth-related controllers/middleware.

---

## Критерий завершения Phase 4

- все ключевые admin use cases доступны через API;
- backend готов обслуживать admin frontend без Blade.

---

# Phase 5. Отдельная админка на Next.js

## Цель

Полностью убрать зависимость от Blade-admin.

## Что делаем

### 5.1. Создать новый проект admin frontend

**Рекомендуемый путь:**
- `D:\Aniyume\aniyume-admin`

**Стек:**
- Next.js
- TypeScript
- Tailwind
- auth guard
- admin API client

---

### 5.2. Реализовать admin MVP экраны

### Первая очередь
- login;
- dashboard;
- anime list/create/edit;
- episodes import/list/edit;
- comments moderation;
- users list/ban/unban;
- import logs;
- audit logs.

### Вторая очередь
- bulk operations;
- filters;
- analytics;
- better UX.

---

### 5.3. Переключить backend с Blade-admin на API-only admin mode

**Что удаляем только после успешной миграции:**
- `aniyume-backend/resources/views/admin/*`
- `aniyume-backend/app/Http/Controllers/Admin/*`
- admin section из `aniyume-backend/routes/web.php`

**Что оставляем:**
- возможно, только health/welcome redirect или вообще минимальный web layer.

---

## Критерий завершения Phase 5

- admin полностью работает на Next.js;
- Blade-admin больше не нужен;
- backend выполняет только API и инфраструктурную роль.

---

# Phase 6. Производительность и оптимизация

## Цель

Сделать backend быстрее и устойчивее.

## Что делаем

### 6.1. Оптимизация каталога anime

**Файл:**
- `aniyume-backend/app/Http/Controllers/Api/V1/AnimeController.php`
- после рефакторинга: query/action classes

**Что проверить и улучшить:**
- сортировки;
- индексы;
- expensive subqueries;
- рекомендации через cache;
- search strategy.

**Дополнительно:**
- составить список необходимых DB индексов;
- оформить отдельной миграцией индексы при необходимости.

---

### 6.2. Кэширование

**Кандидаты на cache:**
- tags list;
- public anime list;
- anime details;
- recommendations;
- banner data;
- dashboard aggregates;
- schedule data.

**Файлы:**
- query classes;
- service classes;
- `config/cache.php`

**Рекомендуемый backend cache:**
- Redis.

---

### 6.3. Импорт и async processing

**Файлы:**
- `aniyume-backend/app/Services/EpisodeImportService.php`
- import jobs/commands
- admin import controllers/api

**Что делать:**
- разбить большой import service на smaller services;
- убрать debug traces;
- добавить better logging;
- вынести тяжелые части в jobs;
- chunk processing вместо `all()` где нужно.

---

### 6.4. Пагинация и лимиты

**Проверить:**
- `UserAnimeListController`
- `EpisodeController`
- `WatchHistoryController`
- admin lists

**Цель:**
- ограничить размеры выдачи;
- добавить безопасные upper bounds;
- унифицировать `per_page` rules.

---

## Критерий завершения Phase 6

- тяжелые endpoints профилированы и улучшены;
- cache strategy внедрена;
- импорт стал безопаснее и чище.

---

# Phase 7. Docker и инфраструктура

## Цель

Сделать проект воспроизводимым и удобным для запуска/демо/защиты.

## Что делаем

### 7.1. Добавить Dockerfile для backend

**Создать:**
- `aniyume-backend/Dockerfile`

**Должен покрывать:**
- PHP runtime;
- composer install;
- artisan bootstrap;
- production/dev profiles по необходимости.

---

### 7.2. Добавить admin и user frontend контейнеры

**Создать:**
- `aniyume/Dockerfile`
- `aniyume-admin/Dockerfile`

---

### 7.3. Добавить docker-compose

**Создать в корне проекта:**
- `D:\Aniyume\docker-compose.yml`

**Сервисы:**
- backend
- frontend
- admin
- db
- redis
- queue
- scheduler
- reverb
- nginx

---

### 7.4. Добавить healthchecks и запуск одной командой

**Цель:**

```bash
docker compose up --build
```

должен поднимать:
- backend;
- frontend;
- admin;
- database;
- queue;
- scheduler;
- reverb.

---

### 7.5. Подготовить deployment docs

**Создать/обновить:**
- `docs/docker-deployment.md`
- `docs/local-setup.md`
- `docs/env-reference.md`

---

## Критерий завершения Phase 7

- проект запускается контейнерами;
- есть воспроизводимый локальный стенд;
- это можно показать на защите.

---

# Phase 8. CI/CD и quality gates

## Цель

Подтвердить качество инженерно.

## Что делаем

### 8.1. Усилить backend pipeline

**Файл:**
- `aniyume-backend/.github/workflows/azure-webapps-php.yml`
- либо новый workflow рядом

**Проверки:**
- composer validate/install;
- php tests;
- pint;
- static analysis;
- route:list;
- schedule:list;
- migrate --pretend.

---

### 8.2. Добавить frontend/admin quality steps

**Проверки:**
- install;
- lint;
- typecheck;
- build.

---

### 8.3. Добавить smoke regression flow

**Сценарии:**
- login;
- public anime catalog;
- add favorite;
- add rating/comment;
- update watch history;
- create/join watch party;
- admin login and basic CRUD.

---

## Критерий завершения Phase 8

- проект проходит автоматические quality gates;
- деплой не идет без базовой проверки качества.

---

# Phase 9. Дипломный пакет артефактов

## Цель

Собрать всё, что нужно для сильной защиты и электронного архива.

## Что делаем

### 9.1. Архитектурные материалы

**Подготовить/обновить:**
- `docs/architecture-overview.md`
- `docs/data-model.md`
- `docs/api-contract.md`
- `docs/sequences.md`
- `docs/deployment.md`
- `docs/security.md`
- `docs/testing-strategy.md`

**Добавить отдельно:**
- `docs/performance-notes.md`
- `docs/admin-api-migration-map.md`
- `docs/backend-module-responsibility-map.md`
- ERD diagram file;
- deployment diagram.

---

### 9.2. Материалы для демонстрации

**Подготовить:**
- `docs/demo-script.md`
- `docs/feature-matrix.md`
- `docs/known-limitations.md`
- `docs/user-guide.md`
- `docs/admin-guide.md`

---

### 9.3. Тестовые протоколы

**Подготовить:**
- результаты backend tests;
- manual scenario checklists;
- screenshots/logs;
- API validation examples;
- docker startup validation.

---

### 9.4. Электронный архив

**Нужно собрать:**
- код;
- диаграммы;
- SQL/migrations;
- инструкции;
- тесты;
- скриншоты;
- ПЗ;
- README.txt с составом архива и инструкцией проверки.

---

## Критерий завершения Phase 9

- проект не просто работает, а полностью готов к защите и передаче в архив.

---

# Приоритетность задач

## P0 — делать сразу
- baseline docs;
- исправление ошибок/LSP;
- security/config cleanup;
- README sync;
- regression checklist.

## P1 — основной backend refactor
- вынос логики из толстых контроллеров;
- action/query structure;
- policies;
- requests/resources standardization.

## P2 — admin separation
- admin API;
- Next admin;
- отказ от Blade.

## P3 — infra and optimization
- Docker;
- CI;
- cache;
- import cleanup;
- performance pass.

## P4 — thesis packaging
- диаграммы;
- протоколы;
- archive package;
- defense script.

---

# Практический порядок старта

Если начинать прямо сейчас, то правильная последовательность такая:

## Шаг 1
Сделать документы baseline:
- `backend-endpoint-inventory.md`
- `admin-api-migration-map.md`
- `backend-module-responsibility-map.md`
- `regression-checklist.md`

## Шаг 2
Починить явные ошибки и опасные config/security места.

## Шаг 3
Начать рефакторинг с:
- `AnimeController`
- `WatchPartyController`
- `FriendshipController`

## Шаг 4
Параллельно спроектировать admin API contract.

## Шаг 5
После стабилизации backend — делать `aniyume-admin` на Next.js.

## Шаг 6
После этого — Docker, CI и финальная дипломная упаковка.

---

# Итог

Этот execution plan нужен, чтобы двигаться **не хаотично**, а по безопасной и профессиональной траектории:

- сначала стабилизировать;
- потом чистить архитектуру;
- потом выносить админку;
- потом инфраструктура;
- потом дипломная упаковка.

Если идти именно так, получится не просто «рефакторинг ради рефакторинга», а **осмысленное доведение проекта до уровня сильной дипломной работы и хорошего инженерного продукта**.
