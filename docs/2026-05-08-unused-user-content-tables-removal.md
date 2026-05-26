# Unused user content tables removal — 2026-05-08

## Цель

Завершить удаление неиспользуемой функциональности пользовательских видео и пользовательских коллекций.

Ранее уже удалены:

- `app/Models/UserVideo.php`
- `app/Models/UserCollection.php`
- relations `User::videos()` и `User::collections()`

Остались только таблицы в базе данных.

## Удаляемые таблицы

Планируется добавить migration, которая удаляет:

- `collection_anime`
- `user_collections`
- `user_videos`

## Почему это безопасно

- нет активных controllers/routes/services для этих сущностей;
- frontend не вызывает API пользовательских видео/коллекций;
- модели удалены и backend tests проходят;
- references по коду остались только в старой migration, которая создавала эти таблицы.

## Риски

Это database-breaking cleanup: если в production есть данные в этих таблицах, они будут удалены.

Перед применением на production рекомендуется:

1. проверить количество записей:

```sql
select count(*) from user_videos;
select count(*) from user_collections;
select count(*) from collection_anime;
```

2. если данные нужны — сделать backup/export;
3. применить migration только после подтверждения, что функциональность не планируется.

## Rollback strategy

`down()` migration восстановит структуру таблиц:

- `user_videos`
- `user_collections`
- `collection_anime`

Важно: rollback восстановит структуру, но не восстановит удаленные данные без backup.

## Проверки после изменения

- `php artisan test`
- `php artisan migrate --pretend` или проверка migration syntax
- `php artisan schedule:list`
- `npm run build`
