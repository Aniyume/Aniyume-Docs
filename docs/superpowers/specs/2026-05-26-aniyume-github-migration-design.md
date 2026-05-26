# Aniyume — миграция на GitHub Organization

**Дата:** 2026-05-26
**Автор:** Tamerlan Zanshugurov (zanshugurov07@gmail.com)
**Статус:** Approved, в исполнении (Фаза 0 завершена)

## 1. Контекст

Сейчас исходный код проекта Aniyume фрагментирован по трём локальным папкам и двум разным GitHub-аккаунтам:

| Папка | GitHub remote | Владелец |
|---|---|---|
| `D:/Aniyume/aniyume` (frontend) | `github.com/KellyHarvestOS/aniyume` | знакомый разработчик |
| `D:/Aniyume/aniyume-backend` (Laravel API) | `github.com/TamerlanWebd/AniYume` | автор |
| `D:/Aniyume/aniyume-admin` (Next.js админка) | — (нет git) | автор |

Дополнительно в корне `D:/Aniyume/` лежат общие артефакты — `docs/` (дипломная документация и technical notes), `infra/docker/`, `docker-compose.yml`.

Состояние веток везде одинаковое — единственная ветка `main`. Workflow с ветками `dev`/`release` не настроен. Деплой через Docker на прод-хост — в планах, но без структуры репозиториев и стратегии веток это невозможно.

## 2. Цели

1. **Единое место для проекта** — все репозитории под одной GitHub Organization `Aniyume`, чтобы команда, дипломная комиссия и будущий деплой видели проект как единое целое.
2. **Чёткое разделение dev и prod** — две защищённые ветки `dev` (рабочая, тестовая) и `release` (production-ready) в каждом репозитории.
3. **Подготовка под Docker deploy** — структура должна позволять автоматически собирать prod-образы из `release` и тестовые из `dev` (в будущем sprint).
4. **Сохранение истории** — текущие коммиты обоих существующих репозиториев должны быть сохранены (через transfer ownership).

## 3. Архитектура

### 3.1. GitHub Organization

Создаётся бесплатная Organization `Aniyume` на github.com. Внутри — четыре репозитория:

| Репо | Содержимое | Стек | Источник |
|---|---|---|---|
| `aniyume-web` | Пользовательский frontend | Next.js 16, React 19, TypeScript, Tailwind 4 | Transfer от `KellyHarvestOS/aniyume`, переименование |
| `aniyume-api` | Backend / API / queue / realtime / admin API | Laravel 12, PHP 8.2+, Sanctum, Reverb | Transfer от `TamerlanWebd/AniYume`, переименование |
| `aniyume-admin-web` | Админ-панель | Next.js | Init с нуля из `D:/Aniyume/aniyume-admin` |
| `aniyume-docs` | Дипломная и архитектурная документация, infra | Markdown + Docker compose | Init с нуля из `D:/Aniyume/docs`, `infra/`, `docker-compose.yml` |

**Почему 4 репозитория, а не один монорепо:** разный lifecycle, разный стек, разные деплои, разные разработчики. Поддерживать монорепо для трёх стеков (PHP, два Next.js) без специнструментов (Nx, Turborepo) дороже, чем держать раздельно.

**Почему отдельный `aniyume-docs`:** дипломная документация обновляется независимо от кода, и доступ к ней может потребоваться комиссии без доступа к исходникам.

### 3.2. Локальная структура после миграции

```
D:/Aniyume/
├── aniyume-web/         (бывшая aniyume,         origin → Aniyume/aniyume-web)
├── aniyume-api/         (бывшая aniyume-backend, origin → Aniyume/aniyume-api)
├── aniyume-admin-web/   (бывшая aniyume-admin,   origin → Aniyume/aniyume-admin-web)
└── aniyume-docs/        (новая,                  origin → Aniyume/aniyume-docs)
    ├── docs/            (move из D:/Aniyume/docs)
    ├── infra/           (move из D:/Aniyume/infra)
    └── docker-compose.yml (move из D:/Aniyume/docker-compose.yml)
```

`.ai-context/` остаётся в `D:/Aniyume/` как локальный кэш для AI-ассистентов, в git не уходит.

### 3.3. Стратегия веток

В каждом из четырёх репозиториев одинаковая модель:

```
release  ──●──────────●──────────●─────  (prod docker image, защищённая)
            ↑          ↑          ↑
            merge      merge      merge
            от dev     от dev     от dev

dev      ──●──●──●──●──●──●──●──●──●──  (default, рабочая)
              ↑     ↑        ↑     ↑
              PR    PR       PR    PR

feature/* fix/* hotfix/* chore/*
```

**Правила:**

- `dev` — default branch в GitHub, все feature-ветки ответвляются от `dev` и вливаются обратно через PR
- `release` — содержит то, что должно быть в проде; обновляется merge'ем из `dev` после ручного тестирования на dev-окружении
- Прямой push в `release` не делаем (когда добавим branch protection — будет блокироваться технически)
- Naming: `feature/<name>`, `fix/<name>`, `hotfix/<name>`, `chore/<name>`
- `main` после миграции упраздняется (переименован в `dev`, плюс создан `release`)

### 3.4. Связь репозиториев

Связь — только через переменные окружения и Docker compose из `aniyume-docs/`. Никаких git submodules или жёстких связей между репозиториями.

```
aniyume-web/.env.local         NEXT_PUBLIC_API_URL=https://api.aniyume.<domain>/api/v1
aniyume-admin-web/.env.local   NEXT_PUBLIC_ADMIN_API_URL=https://api.aniyume.<domain>/api/v1/admin
aniyume-api/.env               CORS_ALLOWED_ORIGINS=https://aniyume.<domain>,https://admin.aniyume.<domain>
```

## 4. План миграции (фазы)

### Фаза 0 — Безопасность (локально, СДЕЛАНО ✅ 2026-05-26)

1. ✅ В `aniyume-backend` нескоммиченные изменения (~50 модифицированных + ~60 untracked файлов) разбиты на 3 логичных коммита:
   - `6a8c3e4` feat: DDD architecture, Docker, CI/CD, code quality tooling
   - `80a5e34` feat: new modules — AI chat, WatchParty, Anilibria, Friendship, Admin API
   - `b845f81` refactor: API V1 cleanup, security hardening, legacy code removal
2. ✅ Push в `TamerlanWebd/AniYume` → `main` (точка отката)
3. ✅ Backup zip: `D:/Aniyume-backup-2026-05-26.zip` (121 MiB, исключены node_modules, vendor, .next, кэши)

### Фаза 1 — GitHub Organization (web, действия пользователя)

1. Открыть https://github.com/organizations/new → создать `Aniyume`, план Free
2. Settings → добавить аватар/описание (опционально)
3. People → пригласить знакомого (KellyHarvestOS) как Member

**Проверка:** `https://github.com/Aniyume` доступен и виден.

### Фаза 2 — Transfer существующих репо (web + локально)

1. Знакомый (KellyHarvestOS) делает: `Settings` → `Transfer ownership` → owner: `Aniyume`
2. После transfer ты переименовываешь `Aniyume/aniyume` → `Aniyume/aniyume-web`
3. Ты делаешь то же для `TamerlanWebd/AniYume` → `Aniyume/aniyume-api`
4. Локально обновить remote:
   ```bash
   cd D:/Aniyume/aniyume
   git remote set-url origin https://github.com/Aniyume/aniyume-web.git
   git fetch origin

   cd D:/Aniyume/aniyume-backend
   git remote set-url origin https://github.com/Aniyume/aniyume-api.git
   git fetch origin
   ```

**Проверка:** `git remote -v` показывает новые URL'ы, `git fetch` работает.

### Фаза 3 — Создание новых репо

1. На GitHub создать пустые `Aniyume/aniyume-admin-web` и `Aniyume/aniyume-docs` (без README/license/.gitignore — наполним сами)
2. Локально для admin:
   ```bash
   cd D:/Aniyume/aniyume-admin
   git init
   git add .
   git commit -m "Initial commit: admin panel scaffold"
   git remote add origin https://github.com/Aniyume/aniyume-admin-web.git
   git branch -M main
   git push -u origin main
   ```
3. Локально создать `aniyume-docs`:
   ```bash
   mkdir D:/Aniyume/aniyume-docs
   # mv D:/Aniyume/docs           D:/Aniyume/aniyume-docs/docs
   # mv D:/Aniyume/infra          D:/Aniyume/aniyume-docs/infra
   # mv D:/Aniyume/docker-compose.yml D:/Aniyume/aniyume-docs/docker-compose.yml
   cd D:/Aniyume/aniyume-docs
   git init && git add .
   git commit -m "Initial commit: docs, infra, docker compose"
   git remote add origin https://github.com/Aniyume/aniyume-docs.git
   git branch -M main && git push -u origin main
   ```

**Проверка:** оба репо доступны на github.com, в каждом есть `main`.

### Фаза 4 — Ветки `dev` / `release` + переименование папок

1. В каждом из 4 репо локально:
   ```bash
   git checkout main && git pull
   git checkout -b release && git push -u origin release
   git checkout -b dev     && git push -u origin dev
   ```
2. На GitHub в каждом репо: `Settings` → `Branches` → `Default branch` → `dev`
3. (опционально позже) удалить `main` — после того как убедимся, что ничего не ломается
4. Переименовать локальные папки:
   - `aniyume` → `aniyume-web`
   - `aniyume-backend` → `aniyume-api`
   - `aniyume-admin` → `aniyume-admin-web`

**Проверка:** в каждом репо default branch = `dev`, есть `release`, локально все ремоуты работают.

## 5. Что вне скоупа этой миграции

Сознательно откладываем, чтобы текущий sprint был сфокусирован:

- **CI/CD pipeline** (GitHub Actions для билдов на dev/release) — отдельный sprint после миграции
- **Docker registry push + auto-deploy на прод-хост** — отдельный sprint
- **Branch protection rules** (запрет push в release, требование PR review) — добавим, когда команда стабилизируется на новой структуре
- **Subdomain DNS / reverse proxy** (`aniyume.<domain>`, `api.<domain>`, `admin.<domain>`) — после того, как будет prod docker host
- **Удаление `main` ветки** — после того, как все привыкнут к `dev`/`release`

## 6. Риски и митигации

| Риск | Митигация |
|---|---|
| Потеря коммитов при transfer ownership | Transfer сохраняет всю историю + есть backup zip от 2026-05-26 |
| Знакомый не сделает transfer вовремя | Альтернатива в плане — mirror clone + push в новый репо |
| Локально обновлённый remote не работает | Проверка `git fetch origin` после каждого `set-url`, в случае ошибки откатываем remote |
| Случайно потеряем данные при move docs/infra/compose | Backup zip и git history `aniyume-backend/docs/` через предыдущие коммиты |
| Зависимости между фронтом и API API URL'ам сломаются | Переменные окружения уже вынесены; в `.env.local` фронта поменяем после Фазы 2 |

## 7. Критерии успеха

После завершения Фазы 4:

- [ ] `https://github.com/Aniyume` существует, в нём 4 репозитория
- [ ] В каждом репозитории default branch = `dev`, существует `release`
- [ ] Локальные папки переименованы по новой схеме
- [ ] `git fetch` и `git push` работают для всех 4 репо
- [ ] Знакомый добавлен как Member в Organization
- [ ] Локальный backup zip сохранён (на случай, если что-то пойдёт не так)
- [ ] Spec-документ существует и закоммичен в `aniyume-docs`
