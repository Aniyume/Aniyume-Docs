# Ratings API tests — 2026-05-08

## Цель

Покрыть тестами пользовательские оценки anime.

Ratings — важная часть community-функциональности:

- пользователь ставит оценку anime;
- может получить свою оценку;
- может видеть список своих оценок;
- может удалить оценку;
- backend должен валидировать score и anime_id.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/RatingsApiTest.php`

Сценарии:

1. Ratings endpoints требуют auth.
2. Пользователь может поставить rating.
3. Повторная оценка обновляет существующий rating, а не создает дубль.
4. Пользователь может получить свой rating по anime.
5. Пользователь может получить список своих ratings.
6. Пользователь может удалить rating.
7. Невалидный score/anime_id возвращает validation error.

## Проверки

```bash
php -l tests/Feature/Api/RatingsApiTest.php
php artisan test --filter=RatingsApiTest
```

## Результат

Ratings tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\RatingsApiTest
✓ ratings endpoints require authentication
✓ user can create rating
✓ repeated rating updates existing record
✓ user can get rating for anime
✓ user can get ratings list
✓ user can delete own rating
✓ user cannot delete another users rating
✓ invalid rating payload returns validation error

Tests: 8 passed (26 assertions)
```

Дополнительно повторно проверены:

```text
FavoritesApiTest — 7 passed (25 assertions)
WatchHistoryApiTest — 7 passed (32 assertions)
UserAnimeListApiTest — 8 passed (30 assertions)
```

## Значение для диплома

Ratings tests показывают, что проект поддерживает не только просмотр, но и community-взаимодействие пользователей с каталогом.
