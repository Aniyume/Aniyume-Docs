# Backend refactoring and thesis compliance master plan

## Цель

Привести проект **Aniyume** к состоянию сильной дипломной работы по специальности «Программное обеспечение» и одновременно подготовить backend к более стабильной, быстрой и поддерживаемой эксплуатации.

Этот план основан на:

- методических указаниях `Методические_указания_ДП_ПО.md`;
- фактическом аудите `aniyume-backend`;
- текущей структуре frontend `aniyume` и документации проекта.

---

## 1. Краткий итог аудита

### Что уже хорошо

Проект уже выглядит как **реальный дипломный fullstack/backend продукт**, потому что в нем есть:

- рабочий Laravel backend;
- Next.js frontend;
- REST API;
- аутентификация и роли;
- каталог anime, эпизоды, комментарии, рейтинги, избранное, история просмотра;
- социальные функции и Watch Party;
- realtime через Reverb;
- админка;
- импорт данных;
- тесты и документация.

### Главные проблемы текущего состояния

1. **Backend не является архитектурно чистым** в строгом смысле.
2. **В проекте смешаны API-backend и Blade-admin** в одном Laravel приложении.
3. **Слишком много бизнес-логики в контроллерах**.
4. **Нет Docker / docker-compose**.
5. **CI/CD слабый** — деплой есть, но quality gates неполные.
6. **Есть технический долг по безопасности, производительности и консистентности слоев**.
7. **Есть рассинхрон между README и реальным кодом** — заявлена clean architecture, но код пока layered monolith с частичным service layer.
8. **Локализация и единообразие API-сообщений непоследовательны**.
9. **Для диплома не хватает полного, формализованного пакета доказательств качества и воспроизводимости**.

---

## 2. Оценка соответствия методическим требованиям дипломной работы

## 2.1. Что соответствует

### Практический результат
Соответствует.

Проект решает прикладную задачу и имеет проверяемый результат:

- backend API;
- frontend;
- база данных;
- бизнес-логика;
- роли пользователей;
- документация;
- тесты;
- сценарии демонстрации.

### Доказательства реализации
Частично соответствует, но надо усилить.

Уже есть:

- исходный код;
- миграции;
- тесты;
- README;
- docs;
- API documentation packages;
- история развития проекта.

Но надо дополнительно стандартизировать:

- архитектурные диаграммы;
- ERD;
- sequence diagrams;
- матрицу требований;
- протоколы тестирования;
- инструкции запуска в Docker;
- демонстрационный архив для защиты.

### Требование к современным инженерным практикам
Частично соответствует.

Есть:

- git;
- README;
- feature tests;
- API docs.

Нет или недостаточно:

- контейнеризация;
- полноценный CI pipeline;
- health checks уровня БД/очередей;
- единая стратегия логирования;
- формальная архитектурная чистота.

## 2.2. Что сейчас не дотягивает до сильной дипломной работы

1. **Архитектура недостаточно чистая для сильной защиты**.
2. **Backend и admin слишком связаны между собой**.
3. **Нет воспроизводимого docker-стенда одной командой**.
4. **Не хватает полного набора обязательных эксплуатационных и тестовых артефактов**.
5. **Часть инфраструктурных настроек небезопасна или слишком грубая**.
6. **Не хватает строгого разграничения application/domain/infrastructure слоев**.

---

## 3. Ключевой стратегический вывод

### Рекомендуемая целевая архитектура

Нужно привести систему к следующей модели:

- **Frontend user app**: Next.js
- **Admin panel**: отдельный Next.js admin client
- **Backend API**: Laravel API-only
- **Realtime layer**: Reverb
- **DB**: PostgreSQL/MySQL
- **Queue/Cache**: Redis
- **Container orchestration**: Docker Compose

### Почему это правильно

Такая схема:

- лучше соответствует современной fullstack архитектуре;
- сильнее смотрится на защите диплома;
- упрощает сопровождение;
- убирает Blade-зависимость из backend;
- позволяет показать разделение ответственности;
- делает backend ближе к clean architecture / API-first подходу.

---

## 4. Основные проблемные зоны

## 4.1. Архитектура

### Симптомы

- толстые контроллеры;
- часть логики в сервисах, часть в контроллерах, часть в моделях;
- не везде применяются FormRequest/Policies/Resources;
- есть смешение web/admin/API ответственности;
- инфраструктурная логика частично живет в middleware.

### Последствия

- сложно тестировать;
- сложно расширять;
- сложно поддерживать;
- трудно аргументировать clean architecture на защите.

## 4.2. Производительность

### Риски

- тяжелая фильтрация и сортировка каталога;
- рекомендации зависят от внешнего API прямо в request flow;
- местами есть риск дорогих запросов;
- часть агрегатов пересчитывается вручную при операциях;
- импорт и admin-операции местами неоптимальны.

## 4.3. Безопасность

### Риски

- слишком широкий CORS;
- жестко заданные env-like значения в конфиге;
- middleware с агрессивной мутацией input;
- нет строгой централизованной security policy;
- неполный rate limiting policy для admin/login и некоторых API зон.

## 4.4. Эксплуатация

### Риски

- нет Docker;
- нет docker-compose;
- нет полноценного локального воспроизводимого стенда;
- CI не выполняет все проверки;
- health check слишком базовый.

## 4.5. Админка

### Текущая проблема

Админка реализована через Blade внутри backend.

### Почему это плохо

- backend перестает быть чистым API;
- смешиваются session-auth web routes и token-auth API;
- сложнее масштабировать и сопровождать;
- слабее выглядит современная fullstack архитектура на защите.

### Цель

- убрать Blade admin из Laravel backend;
- вынести admin panel в **отдельный Next.js application**;
- backend оставить **API-only**.

---

## 5. Приоритетный roadmap рефакторинга

## Phase 0. Freeze и подготовка

### Цель
Зафиксировать текущее состояние и не сломать рабочий функционал.

### Задачи

- создать feature map всех backend возможностей;
- зафиксировать текущие API endpoints;
- зафиксировать текущие admin-сценарии;
- собрать список зависимостей Blade-admin от backend;
- сохранить baseline тестов;
- добавить smoke checklist.

### Артефакты

- API inventory;
- admin feature inventory;
- regression checklist.

---

## Phase 1. Архитектурная стабилизация Laravel backend

### Цель
Сделать backend чище без немедленного полного переписывания.

### Что нужно сделать

#### 1. Ввести четкие слои
Рекомендуемая структура:

- `app/Application` — use cases / services / DTO / actions;
- `app/Domain` — сущности, rules, contracts, policies домена;
- `app/Infrastructure` — integrations, repositories, external services;
- `app/Http` — controllers, requests, resources;
- `app/Admin` — временно до полного удаления Blade-admin.

#### 2. Вынести бизнес-логику из контроллеров
В первую очередь рефакторить:

- `AnimeController`;
- `WatchPartyController`;
- `FriendshipController`;
- `RatingsController`;
- `CommentsController`;
- `UserAnimeListController`.

#### 3. Ввести application services / actions
Примеры:

- `CreateCommentAction`
- `UpdateAnimeStatusAction`
- `StoreRatingAction`
- `SyncWatchHistoryAction`
- `CreateWatchPartyAction`
- `JoinWatchPartyAction`
- `SendFriendRequestAction`
- `ApproveCommentAction`

#### 4. Перенести authorization в Policies
Нужно использовать:

- CommentPolicy
- RatingPolicy
- WatchPartyPolicy
- FriendshipPolicy
- AdminAnimePolicy

#### 5. Стандартизировать API responses
Сделать единый формат ответа:

- `success`
- `message`
- `data`
- `meta`
- `errors`

#### 6. Ввести DTO/Value Objects там, где это полезно
Например:

- profile update payload;
- anime filters;
- watch history payload;
- watch party sync payload.

### Результат фазы

Backend останется рабочим, но станет:

- чище;
- тестируемее;
- понятнее для дипломной защиты.

---

## Phase 2. Декомпозиция admin и уход от Blade

### Цель
Превратить backend в API-only сервис и вынести админку отдельно.

### План

#### Шаг 1. Провести admin API extraction
Для всех admin функций создать JSON API endpoints:

- dashboard metrics;
- users management;
- anime CRUD;
- episode management;
- tags CRUD;
- comments moderation;
- import management;
- audit log browsing.

#### Шаг 2. Создать отдельный Next.js admin app
Варианты:

- `aniyume-admin/` как отдельный проект;
- либо `aniyume` monorepo с отдельным admin route-group.

Рекомендуется для чистоты диплома:

- **отдельный `aniyume-admin`**.

#### Шаг 3. Перевести авторизацию admin на API-based flow
Вместо Blade/session-подхода:

- admin login через API;
- RBAC через roles/permissions;
- admin routes в Next с middleware/guard;
- backend выдает токен или secure cookie scheme.

#### Шаг 4. Удалить Blade views и web admin routes
Удалять только после завершения миграции.

Удаляемое в финале:

- `resources/views/admin/*`
- `app/Http/Controllers/Admin/*` либо переводим в API admin controllers;
- `routes/web.php` admin section;
- лишние session-specific зависимости.

### Результат фазы

- backend станет API-first;
- архитектура станет современнее;
- диплом будет выглядеть сильнее.

---

## Phase 3. Производительность и оптимизация

### Цель
Ускорить backend и снизить нагрузку.

### Основные направления

#### 1. Оптимизация каталога anime

- пересмотреть тяжелые сортировки;
- добавить индексы под фильтры и сортировку;
- вынести рекомендации/сложные выборки в отдельный query service;
- внедрить кэширование популярных публичных endpoints.

#### 2. Кэширование

Кэшировать:

- публичный каталог;
- список тегов;
- детали anime;
- community stats;
- banners;
- рекомендации;
- dashboard aggregates.

Рекомендуемый backend cache:

- Redis.

#### 3. Асинхронность

Вынести из request path:

- тяжелые импорты;
- обновление агрегатов, если возможно;
- внешние API fetch tasks;
- длительные синхронизации.

#### 4. Query optimization

- ликвидировать потенциальные N+1;
- унифицировать eager loading;
- ограничить `per_page` и размеры выборок;
- добавить pagination в проблемные места;
- заменить часть raw SQL более контролируемыми query objects.

#### 5. Агрегаты и counters

- централизовать пересчет рейтинга;
- централизовать counters комментариев;
- при необходимости использовать domain events/listeners.

### Результат фазы

- быстрее публичные endpoints;
- меньше нагрузка на БД;
- более предсказуемое поведение при росте данных.

---

## Phase 4. Безопасность и конфигурация

### Цель
Сделать проект безопаснее и профессиональнее.

### Что исправить

#### High priority

- убрать wildcard CORS для production;
- убрать hardcoded IP/host из config и `.env.example`;
- убрать fingerprint bypass из middleware/config code;
- пересмотреть `SecurityShield` и убрать опасную мутацию input;
- ввести четкий rate limiting policy;
- унифицировать admin/user auth boundaries.

#### Additional

- централизовать validation policy;
- пересмотреть CSP;
- документировать security assumptions;
- разделить dev/stage/prod env profiles.

### Результат фазы

- проект лучше пройдет технические вопросы на защите;
- снизится риск скрытых багов и спорных security решений.

---

## Phase 5. Docker и локальный воспроизводимый стенд

### Цель
Сделать запуск проекта простой и воспроизводимой процедурой.

### Нужно добавить

#### Backend
- `Dockerfile` для Laravel API
- `docker-compose.yml`

#### Infra services
- `nginx`
- `php-fpm` или единый app image
- `postgres` или `mysql`
- `redis`
- `reverb`
- `queue worker`

#### Frontend
- `Dockerfile` для user frontend
- `Dockerfile` для admin frontend

### Минимальная compose-схема

- `frontend`
- `admin`
- `backend`
- `db`
- `redis`
- `queue`
- `scheduler`
- `reverb`
- `nginx`

### Что должно работать одной командой

```bash
docker compose up --build
```

### Дополнительно

- init scripts;
- `.env.example` для каждого сервиса;
- `make` или npm/composer scripts для common workflows;
- healthcheck для контейнеров.

### Результат фазы

Это сильно повышает дипломную ценность проекта:

- воспроизводимость;
- deployability;
- инженерная зрелость.

---

## Phase 6. Тестирование и quality gates

### Цель
Подтвердить качество не словами, а проверками.

### Нужно покрыть

#### Backend tests
- auth;
- public catalog;
- comments;
- ratings;
- favorites;
- watch history;
- user anime list;
- friendship;
- watch party;
- admin API;
- import workflows;
- policies/permissions;
- health endpoints.

#### Frontend tests
- smoke tests;
- critical UI flows;
- admin login + CRUD smoke.

#### Static quality
- `php artisan test`
- `php artisan route:list`
- `php artisan schedule:list`
- `php artisan migrate --pretend`
- `phpstan` или Larastan
- `pint`
- frontend build
- ESLint / TypeScript checks

### CI pipeline должен делать

1. install dependencies;
2. lint;
3. static analysis;
4. tests;
5. build frontend/admin;
6. generate artifacts/docs if needed.

### Результат фазы

- сильные доказательства для диплома;
- меньше регрессий;
- проект выглядит профессионально.

---

## Phase 7. Документация дипломного уровня

### Цель
Подготовить проект так, чтобы он соответствовал требованиям электронного архива и защиты.

### Обязательно подготовить

#### Архитектурные документы
- architecture overview;
- clean architecture scheme;
- ERD;
- sequence diagrams;
- context diagram;
- deployment diagram.

#### Эксплуатационные документы
- backend setup guide;
- docker deployment guide;
- admin guide;
- user guide;
- API contract;
- env configuration guide;
- backup/recovery notes.

#### Документы качества
- testing strategy;
- test protocols;
- known limitations;
- performance notes;
- security notes;
- change log / cleanup report.

#### Для защиты
- demo script;
- feature matrix;
- screenshots;
- список тестовых аккаунтов;
- сценарии демонстрации user/admin/realtime.

### Специально для методички
В архиве должны быть:

- код;
- схемы;
- SQL/migrations;
- тестовые материалы;
- README;
- инструкции запуска;
- доказательства практической реализации.

---

## 6. Матрица приоритетов

## Критично сделать в первую очередь

1. Убрать архитектурный хаос в контроллерах.
2. Подготовить roadmap выноса админки.
3. Добавить Docker Compose.
4. Закрыть security issues в config/middleware.
5. Усилить tests + CI.
6. Подготовить полный комплект артефактов для диплома.

## Средний приоритет

1. Глубокая оптимизация запросов.
2. Redis caching.
3. Event-driven counters/aggregates.
4. Нормализация сообщений API и локализации.
5. Улучшение observability/logging.

## Низкий приоритет

1. Косметическое улучшение нейминга.
2. Дополнительные refactor polish-задачи.
3. Улучшение secondary screens/admin UX.

---

## 7. Конкретный список работ по коду

## Backend architecture

- выделить application layer;
- выделить domain services/use cases;
- сократить контроллеры до orchestration only;
- стандартизировать FormRequests;
- стандартизировать API Resources;
- внедрить Policies/Gates системно;
- унифицировать exception handling.

## Admin extraction

- описать admin use cases;
- сделать admin REST/JSON API;
- создать Next admin client;
- мигрировать экраны;
- отключить Blade-admin.

## Performance

- ревизия `AnimeController`;
- ревизия `WatchPartyController`;
- ревизия import services;
- индексы БД;
- pagination guards;
- caching strategy;
- async jobs cleanup.

## Security

- пересмотреть middleware security stack;
- почистить config/env;
- сузить CORS;
- нормализовать auth boundaries;
- лимиты запросов.

## DevOps

- Dockerfile backend;
- Dockerfile frontend;
- Dockerfile admin;
- docker-compose;
- queue/scheduler/reverb services;
- CI quality pipeline;
- health checks.

## Diploma package

- ERD;
- API docs final version;
- test protocols;
- deployment guide;
- defense demo pack;
- README.txt for archive;
- final ZIP structure.

---

## 8. Рекомендуемый порядок реализации

### Sprint 1 — Stabilize
- аудит и фиксация baseline;
- чистка config/security;
- стандартизация API responses;
- рефакторинг самых тяжелых контроллеров;
- усиление backend tests.

### Sprint 2 — Clean backend core
- use cases / actions;
- policies;
- DTO;
- import/service cleanup;
- optimization pass.

### Sprint 3 — Admin separation
- admin API;
- Next admin MVP;
- миграция основных admin flows.

### Sprint 4 — Infrastructure
- Docker;
- compose;
- CI/CD quality gates;
- health checks;
- deployment docs.

### Sprint 5 — Thesis package
- схемы;
- тестовые протоколы;
- demo script;
- архив проекта;
- финальная защита.

---

## 9. Риски проекта

## Основные риски

1. Слишком большой объем рефакторинга за один этап.
2. Риск сломать рабочие API при переносе логики.
3. Риск затянуть перенос админки.
4. Риск недооценить время на Docker и тесты.
5. Риск не успеть подготовить дипломные артефакты, если сначала делать только код.

## Как снижать риски

- сначала зафиксировать baseline тестами;
- двигаться по фазам;
- не удалять Blade-admin до готовности Next admin;
- после каждого крупного шага прогонять тесты;
- параллельно вести документацию.

---

## 10. Целевое состояние проекта

После выполнения плана проект должен выглядеть так:

### Backend
- Laravel API-only;
- чистая многослойная структура;
- use cases / services / policies;
- хорошие tests;
- Dockerized;
- production-like configs;
- health checks;
- нормальная документация.

### Frontend
- user frontend на Next.js;
- admin frontend на Next.js;
- единый API contract.

### Diploma readiness
- полные доказательства реализации;
- воспроизводимый запуск;
- качественная архитектурная аргументация;
- тестовые протоколы;
- диаграммы;
- инструкции;
- сильная демонстрация на защите.

---

## 11. Итоговый вердикт

### Текущее состояние

Проект **уже можно считать хорошей основой для дипломной работы**, потому что он реально решает прикладную задачу и имеет существенный объем практической реализации.

### Но для уровня сильной защиты и высокого качества нужно обязательно сделать

- архитектурную чистку backend;
- декомпозицию admin в отдельный Next.js клиент;
- Docker-контейнеризацию;
- усиление CI/tests;
- исправление security/config проблем;
- формализацию дипломного пакета доказательств.

### Главная рекомендация

Не начинать сразу «полный переписон». Правильнее идти поэтапно:

1. стабилизация;
2. архитектурная чистка;
3. вынос admin;
4. Docker/CI;
5. дипломная упаковка.

---

## 12. Следующие практические шаги

### Рекомендуемый immediate next step

1. Подтвердить целевую стратегию:
   - оставить Laravel как backend;
   - вынести admin в отдельный Next.js app;
   - перевести backend к API-only;
   - внедрить Docker Compose.
2. После подтверждения — разбить этот master plan на **пошаговый execution plan** с задачами по файлам.
3. Затем выполнять рефакторинг по фазам, начиная с самых рискованных участков.

---

## Appendix A. Статус по ключевым требованиям диплома

| Требование | Статус | Комментарий |
|---|---|---|
| Практический программный результат | Да | Есть fullstack + backend + realtime + admin |
| Исходный код | Да | Есть |
| Инструкция запуска | Частично | Есть, но нужно усилить Docker-вариантом |
| Git / история разработки | Да | Есть |
| README | Да | Есть, но требует синхронизации с реальным кодом |
| API-документация | Да | Есть Scribe/Swagger, надо актуализировать |
| Тестирование | Частично | Есть feature tests, но не все зоны покрыты |
| Архитектурное обоснование | Частично | Нужно усилить clean architecture аргументацию |
| Локализация | Частично | Нужна системность, особенно для UI/admin |
| Контейнеризация | Нет | Нужно добавить |
| Разделение ролей/доступов | Да | Есть, но нужно привести к единому стандарту |
| Доказательства эксплуатационной готовности | Частично | Нужны compose, healthchecks, deployment package |

---

## Appendix B. Приоритетные кандидаты на рефакторинг

### High
- `app/Http/Controllers/Api/V1/AnimeController.php`
- `app/Http/Controllers/Api/V1/WatchPartyController.php`
- `app/Http/Controllers/Api/V1/FriendshipController.php`
- `app/Services/EpisodeImportService.php`
- `app/Http/Middleware/SecurityShield.php`
- `app/Http/Middleware/AntiScraperMiddleware.php`
- `config/cors.php`
- `routes/web.php`
- `resources/views/admin/*`

### Medium
- `app/Http/Controllers/Api/V1/RatingsController.php`
- `app/Http/Controllers/Api/V1/CommentsController.php`
- `app/Http/Controllers/Api/V1/UserAnimeListController.php`
- `app/Services/UserProfileService.php`
- `app/Services/UserStatisticsService.php`
- `.github/workflows/azure-webapps-php.yml`
- `README.md`

### Infra missing
- `Dockerfile`
- `docker-compose.yml`
- CI quality workflow
- health checks beyond `/up`
