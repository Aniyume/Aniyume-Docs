# Backend cleanup notes — 2026-05-08

## Цель

Продолжить архитектурную чистку backend после стабилизации frontend/backend связки:

- отделить временные локальные debug-файлы от production-кода;
- убрать неиспользуемые модели, которые не имеют активных API/routes/services;
- зафиксировать спорные места, которые не стоит удалять без дополнительного решения.

## Временные root-файлы backend

В корне `aniyume-backend/` обнаружены одноразовые скрипты и generated dumps:

- `check_episodes.php`
- `find_anime.php`
- `find_anime_utf8.php`
- `fix_urls.php`
- `test.php`
- `test_import.php`
- `verify_sort.php`
- `verify_urls.php`
- `anilibria_docs.md`
- `anilibria_docs_utf8.md`
- `anime_list.txt`
- `anime_list_utf8.txt`
- `eps_output.txt`
- `eps_utf8.txt`
- `response.txt`
- `schedule.json`
- `timeout_test.txt`

## Решение по временным файлам

Физически файлы пока не удалены, потому что часть из них уже может быть в рабочем дереве как пользовательские untracked/debug artifacts.

Вместо немедленного удаления обновлен `.gitignore`:

- добавлены правила для локальных one-off debug scripts;
- добавлены правила для generated txt/json/md dumps;
- зафиксировано правило: reusable maintenance logic должен жить в `app/Console/Commands`, а не в root php scripts.

Затронутый файл:

- `aniyume-backend/.gitignore`

## Неиспользуемые модели

Проверены references по `app/` и `database/`.

Удалены модели:

- `app/Models/Studio.php`
- `app/Models/Genre.php`
- `app/Models/UserVideo.php`
- `app/Models/UserCollection.php`

Причины:

- `Studio` и `Genre` не имеют migrations для `studios`, `genres`, `anime_studio`, `anime_genre`;
- жанры/категории в текущей архитектуре обслуживаются через `Tag` + `anime_tag`;
- `UserVideo` и `UserCollection` не используются routes/controllers/services;
- references на `UserVideo`/`UserCollection` были только в отношениях `User::videos()` и `User::collections()`.

Также удалены неиспользуемые relations из:

- `app/Models/User.php`

## Что не удалено

### `AnilistImportService`

Не удалялся, хотя прямых references на класс нет.

Причины:

- в проекте есть Anilist-related fields и логика (`external_source = anilist`, `BannerService`, `FillMissingDescriptionsCommand`);
- сервис может быть полезен как fallback/manual import;
- удаление такого сервиса лучше делать отдельным решением после выбора единственного import-source strategy.

### Таблицы `user_videos`, `user_collections`, `collection_anime`

Миграции не переписывались и таблицы не удалялись.

Причина:

- удаление таблиц — breaking DB migration;
- если эта функциональность точно больше не нужна, нужен отдельный migration `drop_user_video_collection_tables`.

## Следующие шаги

1. Если root debug-файлы больше не нужны — удалить их физически отдельной чисткой.
2. Решить судьбу `AnilistImportService`:
   - оставить как documented fallback;
   - или удалить после подтверждения, что импорт только через Shikimori/Anilibria/Kodik.
3. Если пользовательские видео/коллекции точно не нужны — создать migration для удаления таблиц.
4. Добавить README/maintenance policy: все диагностики — artisan commands, не root scripts.
