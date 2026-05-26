# Watch history API tests — 2026-05-08

## Цель

Покрыть тестами сохранение и получение прогресса просмотра.

Watch history — один из центральных сценариев Aniyume:

- пользователь смотрит episode;
- frontend отправляет progress и delta_time;
- backend сохраняет историю просмотра;
- пользователь может продолжить просмотр позже.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/WatchHistoryApiTest.php`

Сценарии:

1. Endpoints требуют auth.
2. Store требует `episode_id`, `progress`, `delta_time`.
3. Пользователь может сохранить прогресс.
4. Повторный store обновляет существующий progress.
5. Можно получить history по anime.
6. Можно получить last watched episode по anime.
7. Пользователь может удалить запись history.

## Проверки

```bash
php -l tests/Feature/Api/WatchHistoryApiTest.php
php artisan test --filter=WatchHistoryApiTest
```

## Результат

Watch history tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\WatchHistoryApiTest
✓ watch history endpoints require authentication
✓ store requires episode progress and delta time
✓ user can store watch progress
✓ repeated store updates existing progress and increments watch time
✓ user can get history by anime
✓ user can get last watched episode
✓ user can delete watch history record

Tests: 7 passed (32 assertions)
```

Дополнительно повторно проверены:

```text
UserAnimeListApiTest — 8 passed (30 assertions)
AuthApiTest — 6 passed (24 assertions)
PublicAnimeApiTest — 6 passed (24 assertions)
HealthAndRoutingTest — 3 passed (5 assertions)
```

## Значение для диплома

Этот тестовый пакет подтверждает персонализированную механику просмотра — одну из главных функций streaming/catalog платформы.
