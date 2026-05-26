# Aniyume GitHub migration plan

## Цель

Безопасно перевести текущий проект Aniyume из локальной рабочей структуры `D:\Aniyume` в новую GitHub-структуру из трех независимых репозиториев:

1. **Aniyume Web** — пользовательский frontend.
2. **Aniyume API** — backend/API.
3. **Aniyume Admin Web** — admin frontend.

В каждом репозитории должны быть две основные ветки:

- `dev` — интеграция и тестирование;
- `release` — стабильная production-линия, откуда происходит выкладка на production Docker host.

Документ не требует менять код, remotes, делать commit или push. Это план миграции и эксплуатации.

## Текущее состояние `D:\Aniyume`

Обнаруженная структура:

```text
D:\Aniyume
├── aniyume/                 # Next.js пользовательский frontend, уже git repo
├── aniyume-backend/         # Laravel backend/API, уже git repo
├── aniyume-admin/           # Next.js admin scaffold, локально, без .git
├── docs/                    # общая проектная документация
├── infra/                   # общая инфраструктура, сейчас nginx config
├── docker-compose.yml       # общий local/dev compose
├── .env.docker.example      # общий docker env example
└── прочие локальные/учебные файлы
```

Технический стек по текущим файлам:

- `aniyume`: Next.js app, `package.json`, `Dockerfile`, `.git`.
- `aniyume-backend`: Laravel 12/PHP 8.2 API, Composer, npm/vite, `Dockerfile`, `.github`, `.git`.
- `aniyume-admin`: Next.js scaffold, `package.json`, `.gitignore`, `.env.example`, без собственного `.git`.
- root `docker-compose.yml`: dev-compose, собирает backend/frontend, db, redis, queue-worker, scheduler, reverb, nginx; admin пока только placeholder.

Важное наблюдение по безопасности миграции:

- `aniyume` и `aniyume-backend` уже имеют собственную git-историю.
- В обоих repo есть локальные незакоммиченные изменения.
- У `aniyume` ветка `main` расходится с `origin/main` (`ahead` и `behind`).
- У `aniyume-backend` ветка `main` локально впереди `origin/main` и также содержит много незакоммиченных/неотслеживаемых файлов.
- `aniyume-admin` пока не является git repo.

Следовательно, миграцию нельзя делать через простое копирование поверх новых репозиториев без предварительного аудита истории и ownership изменений.

## Рекомендуемая финальная GitHub-структура

Рекомендуется создать **GitHub Organization** `Aniyume` или, если имя занято, технически нейтральное имя вроде `aniyume-project`, `aniyume-app`.

Причины выбрать organization вместо personal repos:

- проще управлять доступами разных разработчиков;
- можно настроить teams: `web`, `api`, `admin`, `devops`, `maintainers`;
- проще централизовать GitHub Actions secrets/environments;
- проект выглядит как продукт, а не набор личных репозиториев;
- легче переносить ownership без смены URL каждого repo.

Если проект учебный и пока один владелец, допустимы personal repos, но лучше сразу создать organization, чтобы не мигрировать повторно.

## Репозитории и naming

| Назначение | Display name | Technical repo name | Source path |
|---|---|---|---|
| User frontend | Aniyume Web | `aniyume-web` | `D:\Aniyume\aniyume` |
| Backend/API | Aniyume API | `aniyume-api` | `D:\Aniyume\aniyume-backend` |
| Admin frontend | Aniyume Admin Web | `aniyume-admin-web` | `D:\Aniyume\aniyume-admin` |

Рекомендация: технические имена делать lowercase kebab-case. Display name можно указать в GitHub description/about и README.

Описание репозиториев:

- `aniyume-web`: Public user-facing Next.js frontend for Aniyume.
- `aniyume-api`: Laravel API, queue workers, scheduler and realtime backend for Aniyume.
- `aniyume-admin-web`: Internal/admin Next.js dashboard for content and user management.

## Branch strategy

В каждом repo использовать одинаковую модель:

```text
feature/*, fix/*, chore/*
        ↓ PR
dev
        ↓ release PR после тестирования
release
        ↓ deploy production
production docker host
```

### Ветки

- `dev`: default branch для активной разработки и тестового стенда.
- `release`: protected branch, только проверенные изменения, источник production deploy.
- `feature/<short-name>`: краткоживущие ветки задач от `dev`.
- `hotfix/<short-name>`: срочные исправления от `release`, затем merge обратно в `dev`.

### Почему не `main` как production

Требование проекта явно задает `dev/release`. Поэтому `main` лучше не использовать как активную ветку после миграции. Возможные варианты:

1. На этапе импорта оставить `main` только как legacy/import branch, затем сделать `dev` default.
2. Создать `dev` и `release` от текущей стабильной точки, после чего `main` не использовать и защитить/архивировать.
3. Если GitHub требует default branch при создании repo, создать `dev` первой или переключить default branch на `dev` сразу после импорта.

Рекомендация: default branch = `dev`, production branch = `release`.

## Branch protection

Для всех трех repo:

### `dev`

- Require pull request before merging.
- Require status checks: lint/build/test в зависимости от repo.
- Require branches to be up to date before merging, если CI не слишком медленный.
- Block direct pushes для всех, кроме maintainers или полностью для всех.
- Allow squash merge для feature branches.

### `release`

- Require pull request before merging.
- Require approvals: минимум 1, лучше 2 для API.
- Require successful CI.
- Require deployment approval через GitHub Environments для production.
- Restrict who can push.
- Block force pushes.
- Block deletions.
- Require signed tags/releases желательно, но не обязательно на первом этапе.

## Что куда переносить

### `D:\Aniyume\aniyume` → `aniyume-web`

Переносится весь frontend repo как корень нового репозитория:

```text
aniyume-web/
├── app/
├── components/
├── contexts/
├── hooks/
├── lib/
├── public/
├── types/
├── Dockerfile
├── package.json
├── package-lock.json
├── next.config.ts
├── tsconfig.json
└── README.md
```

Не переносить:

- `node_modules/`;
- `.next/`;
- `tsconfig.tsbuildinfo`;
- локальные `.env`, если появятся;
- прочие build/cache artifacts.

Проверить `.gitignore`, чтобы эти директории не попадали в repo.

### `D:\Aniyume\aniyume-backend` → `aniyume-api`

Переносится backend repo как корень нового репозитория:

```text
aniyume-api/
├── app/
├── bootstrap/
├── config/
├── database/
├── public/
├── resources/
├── routes/
├── storage/              # только нужная структура, не runtime файлы
├── tests/
├── Dockerfile
├── artisan
├── composer.json
├── composer.lock
├── package.json
├── package-lock.json
└── README.md
```

Не переносить:

- `.env`;
- `vendor/`;
- `node_modules/`;
- `.phpunit.result.cache`;
- runtime/log/cache из `storage/`, кроме стандартных `.gitignore`/структуры;
- generated docs/cache, если они не являются осознанной частью репозитория.

Особенно важно: текущий `D:\Aniyume\aniyume-backend\.env` не должен попасть в GitHub. Проверить, что он не tracked.

### `D:\Aniyume\aniyume-admin` → `aniyume-admin-web`

Так как `.git` в admin scaffold не обнаружен, история, вероятно, отсутствует. Создать новый repo с текущим scaffold как initial import.

```text
aniyume-admin-web/
├── src/
├── next.config.mjs
├── package.json
├── package-lock.json
├── tsconfig.json
├── .env.example
├── .gitignore
└── README.md
```

Не переносить:

- `node_modules/`;
- `.next/`;
- локальные env-файлы.

### Root orchestration

Текущие root-файлы:

- `D:\Aniyume\docker-compose.yml`;
- `D:\Aniyume\.env.docker.example`;
- `D:\Aniyume\infra/docker/nginx/default.conf`.

Это не принадлежит полностью ни одному из трех application repos. Есть три варианта.

#### Вариант A — отдельный infra repo, рекомендуется для production

Создать четвертый приватный repo:

- Display name: **Aniyume Infra**
- Technical name: `aniyume-infra`

Содержимое:

```text
aniyume-infra/
├── compose/
│   ├── docker-compose.dev.yml
│   ├── docker-compose.release.yml
│   └── .env.example
├── nginx/
│   └── default.conf
├── scripts/
│   ├── deploy-release.sh
│   └── rollback.sh
└── docs/
    └── operations.md
```

Плюсы: чистая ответственность, production deploy не завязан на web/api/admin repo. Минусы: появляется четвертый repo, хотя исходное требование говорит о трех основных repo.

#### Вариант B — infra в `aniyume-api`, допустимо на первом этапе

Разместить production compose и nginx рядом с API, потому что API управляет db/redis/queue/scheduler/reverb.

Плюсы: только три repo. Минусы: web/admin deploy config становится зависим от API repo.

#### Вариант C — infra в docs/ops внутри одного repo

Держать orchestration в `aniyume-api/docs/ops` или `aniyume-web/docs/ops`. Это менее чисто и со временем станет неудобно.

Рекомендация: для практичности начать с трех application repos + временно хранить root orchestration в `aniyume-api/infra` или оставить локально до создания `aniyume-infra`. Для production зрелости лучше создать отдельный `aniyume-infra` после стабилизации трех основных repos.

## Как не потерять git-историю

### Главное правило

Не создавать новые пустые repo и не копировать туда файлы вручную для `aniyume` и `aniyume-backend`, если нужно сохранить историю. Нужно переносить существующие git repositories.

### Для `aniyume-web`

Текущий `D:\Aniyume\aniyume` уже git repo. Безопасная схема:

1. Зафиксировать состояние у текущего владельца/разработчика: кто владеет `origin`, какие ветки есть, какие изменения локальные.
2. Не менять текущий `origin`, пока не согласована миграция.
3. Создать новый GitHub repo `aniyume-web` без README/license/gitignore, чтобы он был пустым.
4. Сделать mirror/push существующей истории в новый repo из отдельного временного clone или после согласования remote.
5. Создать `dev` от согласованной интеграционной точки.
6. Создать `release` от последнего production-stable commit, а не автоматически от самой новой локальной рабочей копии.

Если история diverged (`ahead/behind`), сначала нужно решить, какие commits из локального `main` и `origin/main` должны войти в `dev`.

### Для `aniyume-api`

Текущий `D:\Aniyume\aniyume-backend` уже git repo. Схема аналогична:

1. Проверить текущий `origin` и владельца backend repo.
2. Проверить, какие локальные commits уже не отправлены.
3. Проверить незакоммиченные изменения и неотслеживаемые файлы.
4. Согласовать, что является стабильной базой для `release`.
5. Перенести историю в `aniyume-api` через mirror или обычный push всех нужных веток/tags.

### Для `aniyume-admin-web`

Admin scaffold не имеет git-истории, поэтому:

1. Создать новый repo `aniyume-admin-web`.
2. Импортировать scaffold первым commit.
3. Создать `dev` как default.
4. Создать `release` от `dev` только после первого успешного build и smoke-test.

### Команды только как пример для будущего выполнения

Не выполнять до согласования ownership и freeze window.

```bash
# Пример mirror-переноса истории существующего repo
git clone --mirror <old-repo-url> aniyume-web.git
cd aniyume-web.git
git remote set-url origin git@github.com:Aniyume/aniyume-web.git
git push --mirror
```

Если работа идет из локального repo и remote менять нельзя, использовать дополнительный remote с временным именем, например `github-new`, но только после согласования:

```bash
git remote add github-new git@github.com:Aniyume/aniyume-web.git
git push github-new --all
git push github-new --tags
```

В рамках текущей аналитической задачи remotes не менять.

## Пошаговый migration plan

### Phase 0 — Freeze and inventory

1. Объявить короткое migration freeze window.
2. Попросить всех разработчиков выполнить локальный backup/clone.
3. Собрать список текущих repo/remotes/branches/tags по web и API.
4. Зафиксировать, какие локальные изменения у кого не закоммичены.
5. Проверить наличие secrets:
   - `.env`;
   - credentials;
   - private keys;
   - tokens;
   - production passwords.
6. Убедиться, что `node_modules`, `vendor`, `.next`, caches не попадут в перенос.

### Phase 1 — Создать GitHub Organization и repos

1. Создать GitHub Organization `Aniyume` или `aniyume-project`.
2. Создать repos:
   - `aniyume-web`;
   - `aniyume-api`;
   - `aniyume-admin-web`.
3. Создавать repos пустыми: без auto README, license, gitignore, чтобы не создавать конфликтующий initial commit.
4. Настроить teams и permissions:
   - maintainers: admin;
   - developers: write;
   - deploy bot/GitHub Actions: scoped access.

### Phase 2 — Перенести web

1. Выбрать canonical source: текущий repo разработчика frontend или `D:\Aniyume\aniyume`.
2. Если локальный `main` diverged from `origin/main`, сначала создать integration branch и разрешить расхождения.
3. Перенести историю в `aniyume-web`.
4. Создать ветку `dev` от актуальной интеграционной точки.
5. Создать ветку `release` от стабильной точки, которая точно собирается и может быть задеплоена.
6. Сделать `dev` default branch.
7. Настроить CI: `npm ci`, `npm run lint`, `npm run build`.

### Phase 3 — Перенести API

1. Выбрать canonical source: текущий backend repo или `D:\Aniyume\aniyume-backend`.
2. Проверить незакоммиченные и untracked изменения, отдельно решить их судьбу.
3. Убедиться, что `.env` не tracked и не будет опубликован.
4. Перенести историю в `aniyume-api`.
5. Создать `dev` от актуальной интеграционной точки.
6. Создать `release` от стабильной точки.
7. Сделать `dev` default branch.
8. Настроить CI:
   - `composer install --no-interaction --prefer-dist`;
   - `php artisan test`;
   - `composer analyse` или `phpstan`, если используется;
   - optional: `npm ci && npm run build` для vite assets.

### Phase 4 — Создать admin repo

1. Создать `aniyume-admin-web`.
2. Импортировать содержимое `D:\Aniyume\aniyume-admin`, исключив `node_modules` и `.next`.
3. Создать `dev` как default.
4. Настроить CI: `npm ci`, `npm run build`; lint script нужно привести в рабочее состояние отдельно, так как текущий Next 14 scaffold использует `next lint`, что может потребовать проверки совместимости.
5. Создать `release` после первого успешного CI.

### Phase 5 — Branch protection и environments

1. Настроить protection для `dev` и `release` в каждом repo.
2. Создать GitHub Environments:
   - `development` для deploy из `dev` на test/staging;
   - `production` для deploy из `release`.
3. В `production` включить required reviewers.
4. Добавить secrets отдельно по repo/environment, не хранить в коде.

### Phase 6 — Deploy flow

Рекомендуемый production flow:

1. Разработчик делает PR в `dev`.
2. CI проверяет изменения.
3. После merge в `dev` происходит deploy на test/staging docker host или ручной staging deploy.
4. После проверки создается PR `dev` → `release`.
5. CI собирает production images.
6. После approval GitHub Actions публикует Docker images с тегами:
   - `aniyume-web:<sha>` и `aniyume-web:release`;
   - `aniyume-api:<sha>` и `aniyume-api:release`;
   - `aniyume-admin-web:<sha>` и `aniyume-admin-web:release`.
7. Production Docker host подтягивает images по immutable SHA tag или release tag.
8. Выполняется rolling/recreate deploy через compose.
9. Для API migrations выполняются controlled steps:
   - backup DB;
   - `php artisan migrate --force`;
   - healthcheck;
   - rollback plan при ошибке.

### Production Docker host strategy

Лучше не собирать production код прямо на сервере через `git pull`. Безопаснее:

- GitHub Actions build images;
- push в GHCR или Docker Registry;
- production host делает `docker compose pull && docker compose up -d`;
- host хранит только compose/env/secrets, но не исходники.

Пример naming images:

```text
ghcr.io/aniyume/aniyume-web:<git-sha>
ghcr.io/aniyume/aniyume-api:<git-sha>
ghcr.io/aniyume/aniyume-admin-web:<git-sha>
```

Для rollback:

- хранить предыдущий successful image tag;
- compose должен позволять быстро вернуть tag;
- DB migrations должны быть backward-compatible или иметь ручной rollback plan.

## Как поступить с docs

Текущий `D:\Aniyume\docs` содержит общую документацию по архитектуре, API, деплою, тестированию, admin, дипломной готовности.

### Рекомендация на ближайший этап

Не дробить docs механически сразу. Сделать `aniyume-api` canonical docs source для общей backend/API/architecture/deployment документации, а в web/admin repos оставить короткие README со ссылкой на canonical docs.

Почему API repo как canonical source:

- API содержит центральную доменную модель и контракты;
- deployment сейчас больше завязан на backend/db/redis/queue/reverb;
- многие docs уже описывают backend/API/tests;
- меньше риск рассинхронизации на старте.

### Более зрелый вариант

После стабилизации создать отдельный repo:

- Display name: **Aniyume Docs**
- Technical name: `aniyume-docs`

Туда перенести cross-repo documentation:

- architecture overview;
- API contract;
- deployment;
- security;
- user/admin guides;
- diagrams;
- thesis/demo docs.

В application repos оставить только локальные README и component-specific docs.

### Что делать с docs при миграции сейчас

1. Разделить документы по ownership:
   - API-specific → `aniyume-api/docs`;
   - web-specific → `aniyume-web/docs`;
   - admin-specific → `aniyume-admin-web/docs`;
   - cross-cutting → временно `aniyume-api/docs` или будущий `aniyume-docs`.
2. Внести в README каждого repo ссылку на canonical docs.
3. Не переносить docs в несколько repo копированием без владельца — это создаст рассинхронизацию.

## Безопасный порядок перехода при распределенной истории

1. Назначить одного migration owner.
2. Запретить force push и самостоятельную смену remotes на время миграции.
3. Собрать от каждого разработчика:
   - URL текущего repo;
   - список локальных веток;
   - `git status`;
   - список незапушенных commits;
   - есть ли незакоммиченные изменения.
4. Для каждого codebase выбрать canonical history source.
5. Если есть diverged histories, не перезаписывать одну другой. Создать временную integration branch и смержить/черри-пикнуть нужные commits.
6. Перед push в новые GitHub repos создать backup tags/branches:
   - `backup/pre-migration-web-YYYYMMDD`;
   - `backup/pre-migration-api-YYYYMMDD`.
7. Проверить CI/build/test на `dev`.
8. Создать `release` только от проверенного commit.
9. После миграции старые repos перевести в read-only/archive или оставить с README, указывающим новый canonical repo.

## Риски

| Риск | Вероятность | Влияние | Mitigation |
|---|---:|---:|---|
| Потеря git-истории при копировании файлов | Средняя | Высокое | Переносить существующие repos через git push/mirror, не через manual copy |
| Публикация secrets из `.env` | Средняя | Критическое | Secret scan, `.gitignore`, ручная проверка tracked files перед push |
| Diverged frontend history | Высокая | Среднее/высокое | Integration branch, согласование commits, backup branches |
| Незакоммиченные backend изменения потеряются | Высокая | Высокое | Inventory, stash/patch/branch у владельца до миграции |
| `release` создан от нестабильной локальной версии | Средняя | Высокое | Создавать `release` только после CI и smoke-test |
| Docs рассинхронизируются между repos | Средняя | Среднее | Один canonical docs source, ссылки вместо копий |
| Production deploy из `dev` по ошибке | Средняя | Высокое | Deploy только из `release`, protected environment, reviewers |
| DB migrations ломают rollback | Средняя | Высокое | Backward-compatible migrations, backup перед migrate, rollback runbook |
| Docker compose зависит от относительных путей старого monorepo | Высокая | Среднее | Перепроектировать compose под images/registry или infra repo |
| Admin repo содержит build artifacts | Средняя | Низкое/среднее | Исключить `.next`, `node_modules`, проверить `.gitignore` |

## Итоговая рекомендация

1. Создать GitHub Organization `Aniyume`.
2. Создать три application repos:
   - `aniyume-web`;
   - `aniyume-api`;
   - `aniyume-admin-web`.
3. В каждом repo использовать `dev` как default branch и `release` как production branch.
4. Переносить `aniyume` и `aniyume-backend` с сохранением существующей git-истории.
5. `aniyume-admin` импортировать как новый repo, так как git-история локально не обнаружена.
6. Production deploy делать только из `release`, через Docker images и GitHub Environments approval.
7. Docs временно держать в одном canonical source, предпочтительно `aniyume-api/docs`, либо позднее вынести в `aniyume-docs`.
8. Root orchestration не размазывать по application repos; в идеале вынести в `aniyume-infra`, а на первом этапе допустимо временно держать deploy docs/config рядом с API.
