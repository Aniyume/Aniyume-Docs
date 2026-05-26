# Favorites API tests — 2026-05-08

## Цель

Покрыть тестами избранное anime.

Favorites — простой, но важный пользовательский сценарий:

- пользователь добавляет anime в избранное;
- видит список избранного;
- проверяет, находится ли anime в избранном;
- удаляет anime из избранного.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/FavoritesApiTest.php`

Сценарии:

1. Favorites endpoints требуют auth.
2. Пользователь может добавить anime в избранное.
3. Повторное добавление не создает дубль.
4. Пользователь может получить список избранного.
5. Пользователь может проверить наличие anime в избранном.
6. Пользователь может удалить anime из избранного.
7. Невалидный `anime_id` возвращает validation error.

## Проверки

```bash
php -l tests/Feature/Api/FavoritesApiTest.php
php artisan test --filter=FavoritesApiTest
```

## Результат

Favorites tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\FavoritesApiTest
✓ favorites endpoints require authentication
✓ user can add anime to favorites
✓ duplicate favorite returns conflict
✓ user can get favorites list
✓ user can check favorite status
✓ user can remove anime from favorites
✓ invalid anime id returns validation error

Tests: 7 passed (25 assertions)
```

Дополнительно повторно проверены:

```text
WatchHistoryApiTest — 7 passed (32 assertions)
UserAnimeListApiTest — 8 passed (30 assertions)
AuthApiTest — 6 passed (24 assertions)
PublicAnimeApiTest — 6 passed (24 assertions)
```

## Значение для диплома

Favorites tests дополняют пользовательский контур после auth, anime list и watch history.
