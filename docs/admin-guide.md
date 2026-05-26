# Admin guide

## Назначение

Admin panel предназначена для управления каталогом, пользователями, комментариями, импортами и аудитом.

## Доступ

Admin routes защищены middleware:

```php
['auth', 'admin']
```

Пользователь должен иметь role:

```text
admin
```

## Основные разделы

### Dashboard

Назначение:

- обзор состояния системы;
- быстрые ссылки на management sections;
- статистика, если доступна.

### Anime management

Назначение:

- просмотр списка anime;
- редактирование metadata;
- управление статусом/типом/постерами;
- запуск import/update operations.

### Episode management

Назначение:

- просмотр episodes;
- импорт episodes для anime;
- bulk import;
- удаление/обновление episode data.

Исправленная логика:

- `EpisodesImportJob` существует;
- `bulkImport()` реализован;
- import pipeline использует `EpisodeImportService`.

### Tag management

Назначение:

- управление tags/genres;
- tags используются вместо legacy `Genre` model.

### User management

Назначение:

- просмотр пользователей;
- управление статусами пользователей;
- проверка roles.

### Comment moderation

Назначение:

- просмотр comments;
- модерация/удаление problematic content.

### Import management

Назначение:

- запуск anime import;
- запуск episode import;
- просмотр import logs.

Канонические команды:

```bash
php artisan import:anime
php artisan import:episodes --limit=200
php artisan episodes:sync-ongoing --days=1
```

### Audit logs

Назначение:

- просмотр административных/системных событий;
- анализ действий в панели.

## Scheduler

Актуальная конфигурация находится в:

```text
routes/console.php
```

Активные задачи:

```text
0 3   * * *  php artisan import:anime
0 *   * * *  php artisan import:episodes --limit=200
0 */6 * * *  php artisan episodes:sync-ongoing --days=1
```

## Queue worker

Для production imports/jobs нужен queue worker:

```bash
php artisan queue:work --tries=3 --timeout=300
```

## Production checklist для admin

- [ ] admin user создан;
- [ ] role `admin` назначена;
- [ ] routes защищены `auth` + `admin`;
- [ ] queue worker запущен;
- [ ] scheduler cron настроен;
- [ ] import commands работают;
- [ ] audit logs доступны;
- [ ] comment moderation доступна.

## Demo на защите

Показать:

1. вход под admin;
2. dashboard;
3. anime management;
4. episode import;
5. comment moderation;
6. users/roles;
7. schedule/import commands в terminal/docs.
