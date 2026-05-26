# User anime list API tests — 2026-05-08

## Цель

Покрыть тестами пользовательский список просмотра anime.

Это один из ключевых продуктовых сценариев:

- пользователь добавляет anime в список;
- меняет статус просмотра;
- видит список по статусам;
- обновляет количество просмотренных эпизодов;
- может убрать anime из списка.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/UserAnimeListApiTest.php`

Сценарии:

1. Private endpoints требуют auth.
2. Пользователь может установить статус anime.
3. Пользователь может получить свой статус anime.
4. `GET /my-anime-list/{status}` возвращает anime нужного статуса.
5. `status = not_watching` удаляет anime из списка.
6. Пользователь может обновить `episodes_watched`.
7. Невалидный статус возвращает validation error.

## Проверки

```bash
php -l tests/Feature/Api/UserAnimeListApiTest.php
php artisan test --filter=UserAnimeListApiTest
```

## Результат

User anime list tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\UserAnimeListApiTest
✓ user anime list requires authentication
✓ user can set anime status
✓ user can get anime status
✓ not listed anime returns not watching status
✓ user can get list filtered by status
✓ not watching status removes anime from list
✓ user can update episodes watched
✓ invalid status returns validation error

Tests: 8 passed (30 assertions)
```

Дополнительно повторно проверены:

```text
AuthApiTest — 6 passed (24 assertions)
PublicAnimeApiTest — 6 passed (24 assertions)
HealthAndRoutingTest — 3 passed (5 assertions)
```

## Значение для диплома

Эти тесты подтверждают, что приложение поддерживает персонализированный пользовательский сценарий, а не только публичный каталог и авторизацию.
