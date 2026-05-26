# Watch Party API tests — 2026-05-08

## Цель

Покрыть тестами REST-часть Watch Party.

Watch Party — одна из самых сильных функций Aniyume для дипломной защиты:

- совместный просмотр;
- комнаты по коду;
- участники;
- чат;
- синхронизация плеера;
- приглашения друзей;
- закрытие комнаты.

Realtime delivery через Reverb проверяется отдельными manual/demo сценариями, но REST-слой должен быть покрыт feature tests.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/WatchPartyApiTest.php`

Сценарии:

1. Watch Party endpoints требуют auth.
2. Пользователь может создать room.
3. Пользователь может получить room по code.
4. Другой пользователь может join room.
5. Участник может отправить message.
6. Участник может получить messages.
7. Host может отправить sync event.
8. Участник может leave room.
9. Host может close room.

## Проверки

```bash
php -l tests/Feature/Api/WatchPartyApiTest.php
php artisan test --filter=WatchPartyApiTest
```

## Результат

Watch Party REST tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\WatchPartyApiTest
✓ watch party endpoints require authentication
✓ user can create watch party room
✓ user can show active room
✓ another user can join room
✓ participant can send and fetch messages
✓ non participant cannot send message
✓ host can sync player state
✓ non host cannot sync player state
✓ participant can leave room
✓ host can close room
✓ non host cannot close room

Tests: 11 passed (49 assertions)
```

Дополнительно повторно проверены:

```text
CommentsApiTest — 8 passed (30 assertions)
RatingsApiTest — 8 passed (26 assertions)
FavoritesApiTest — 7 passed (25 assertions)
```

## Значение для диплома

Этот тестовый пакет подтверждает, что самая демонстрационная realtime-функциональность имеет надежный backend REST фундамент.
