# Demo script

## Цель

Этот сценарий нужен для уверенной демонстрации Aniyume на защите дипломной работы.

Рекомендуемый формат демонстрации: 7–12 минут.

## Подготовка перед показом

Проверить, что запущены:

- preferred local/demo baseline: `docker compose up --build -d` from repository root;
- database (`db`) and Redis (`redis`);
- backend Laravel server (`backend`), frontend Next.js server (`frontend`) and nginx reverse proxy (`nginx`);
- queue worker, если демонстрируются queued imports;
- Reverb server, если демонстрируется realtime Watch Party;
- scheduler, если показывается operational часть.

Быстрые health checks:

```bash
docker compose ps
curl.exe --max-time 60 http://localhost:8088/nginx-health
curl.exe --max-time 60 http://localhost:8000/up
```

Основная точка входа для Docker demo: `http://localhost:8088`.

Подготовить аккаунты:

- обычный пользователь #1;
- обычный пользователь #2 для Watch Party;
- admin user.

Подготовить данные:

- несколько anime в каталоге;
- несколько episodes;
- tags/genres;
- хотя бы один anime с доступным player source;
- несколько comments/ratings для демонстрации community layer.

Если показывается AI сценарий:

- подготовить обычный пользовательский token/session;
- проверить AI env/provider или заранее принять fallback/stub поведение как честный backup;
- подготовить 2–3 безопасных prompt'а в домене AniYume.

## 1. Вступление

Коротко описать проект:

> Aniyume — fullstack web-платформа для anime-каталога, просмотра эпизодов, пользовательских списков, истории просмотра, community-функций и совместного просмотра через Watch Party.

Ключевые технологии:

- Next.js frontend;
- Laravel backend;
- REST API;
- Sanctum auth;
- Reverb realtime;
- scheduler/import jobs;
- admin panel.

## 2. Каталог и поиск

Действия:

1. Открыть главную страницу.
2. Перейти в catalog/filter page.
3. Показать список anime.
4. Использовать поиск.
5. Использовать фильтр по status/year/genre.
6. Открыть страницу anime.

Что сказать:

- каталог загружается через публичный API;
- frontend ходит через unified proxy `/api/external`;
- backend поддерживает pagination, filtering, sorting.

## 3. Страница anime и episodes

Действия:

1. Открыть anime details.
2. Показать описание, постер/banner, теги.
3. Показать список episodes.
4. Открыть player.

Что сказать:

- episodes подтягиваются из backend;
- player/P2P часть загружается client-side, чтобы не ломать SSR build;
- backend имеет endpoint для player sources.

## 4. Авторизация

Действия:

1. Открыть login/register.
2. Войти под обычным пользователем.
3. Открыть профиль.

Что сказать:

- backend выдает Sanctum Bearer token;
- frontend хранит token и прокидывает его через API proxy;
- private endpoints защищены `auth:sanctum`.

## 5. User anime list

Действия:

1. На странице anime выбрать статус `watching` или `planned`.
2. Открыть bookmarks/profile list.
3. Показать, что anime появился в списке.
4. Убрать anime из списка через `not_watching`.

Что сказать:

- пользовательский список хранится в pivot table `anime_user`;
- backend покрыт feature tests для статусов и прогресса episodes watched.

## 6. Watch history

Действия:

1. Запустить episode.
2. Подождать несколько секунд.
3. Показать сохранение прогресса.
4. Обновить страницу и показать восстановление/last watched episode, если UI это отображает.

Что сказать:

- frontend отправляет `episode_id`, `progress`, `delta_time`;
- backend сохраняет watch history;
- есть tests на сохранение, повторное обновление, last watched episode и удаление history.

## 7. Community layer: favorites, ratings, comments

Действия:

1. Добавить anime в favorites.
2. Поставить rating.
3. Оставить comment.
4. Показать comments list.

Что сказать:

- community-функции закрывают взаимодействие пользователя с каталогом;
- есть tests на favorites, ratings и comments;
- чужие comments/ratings защищены от изменения.

## 8. Watch Party

Действия:

1. Под пользователем #1 создать Watch Party room.
2. Открыть вторую сессию/браузер под пользователем #2.
3. Join room по code/link.
4. Отправить chat message.
5. Синхронизировать play/pause/seek с host.
6. Закрыть комнату host-ом.

Что сказать:

- REST layer отвечает за room lifecycle, participants, messages, sync commands;
- Reverb обеспечивает realtime delivery;
- host-only actions валидируются backend-ом;
- WatchParty REST покрыт feature tests.

## 9. Admin panel

Действия:

1. Войти под admin.
2. Открыть dashboard.
3. Показать anime management.
4. Показать episode management/import.
5. Показать comment moderation/users/audit logs, если доступно.

Что сказать:

- admin routes защищены `auth` + `admin` middleware;
- также существует JSON admin API под `/api/v1/admin/*` для перехода к отдельному Next.js admin client;
- отдельный `aniyume-admin` scaffold уже есть, но auth/session flow пока transitional и не заменяет полностью legacy Blade-admin;
- import jobs/scheduler автоматизируют обновление каталога;
- audit/admin layer важен для production-like системы.

## 10. AI module

Действия:

1. Войти под пользователем.
2. Открыть AI chat UI, если он доступен в текущем frontend build, или показать API endpoint/code/docs как backend capability.
3. Отправить безопасный доменный запрос, например: рекомендация anime по предпочтениям или объяснение функции платформы.
4. Показать, что AI работает через backend endpoint и не требует прямого доступа frontend к provider secret.

Что сказать:

- AI модуль реализован на backend и доступен через `POST /api/v1/ai/chat`;
- есть session history endpoints;
- есть role/policy, throttling, tool allowlist и output guardrails;
- provider/API-key/network availability относится к runtime окружению, поэтому для защиты нужен заранее проверенный env или backup через документацию/код.

Backup:

- если внешний AI provider недоступен, показать fallback/stub behavior, структуру AI модулей и known limitations;
- не вводить и не показывать реальные секреты/API keys.

## 11. Документация и тесты

Показать каталог `docs/`:

- `architecture-overview.md`;
- `api-contract.md`;
- `data-model.md`;
- `sequences.md`;
- `security.md`;
- `deployment.md`;
- `testing-strategy.md`;
- `thesis-readiness-plan.md`;
- `feature-matrix.md`;
- `known-limitations.md`;
- `docker-setup.md`;
- `final-thesis-readiness-audit.md`.

Показать тесты:

```bash
php artisan test
npm run build
```

Что сказать:

- проект не только реализован, но и инженерно оформлен;
- есть API contracts, ERD, sequence diagrams, deployment/security notes;
- ключевые backend APIs покрыты feature tests.

## 12. Финальное резюме

Коротко завершить:

> В результате разработана fullstack-платформа с каталогом anime, пользовательскими сценариями просмотра, community-функциями, realtime Watch Party, admin/admin API layer, AI backend module, background imports, Docker demo stack, документацией и автоматическими тестами ключевых backend API.

## Backup plan

Если realtime/Reverb не работает на защите:

- показать REST Watch Party tests;
- показать `sequences.md`;
- показать `WatchPartyApiTest`;
- объяснить, что realtime delivery зависит от websocket server/reverse proxy, а REST foundation покрыт тестами.

Если external video source недоступен:

- показать catalog/details/user flows;
- показать player sources endpoint;
- объяснить зависимость от внешних video providers.

Если AI provider недоступен:

- показать AI endpoint/session routes и модульную структуру backend;
- показать safety/role/tool notes в final audit;
- объяснить, что внешний provider зависит от runtime env/API key/network, а секреты на защите не демонстрируются.
