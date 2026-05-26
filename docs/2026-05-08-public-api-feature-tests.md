# Public API feature tests — 2026-05-08

## Цель

Добавить первые полноценные backend Feature tests для публичного API каталога.

Это повышает качество проекта для дипломной работы, потому что показывает:

- проверяемость REST API;
- стабильность публичных endpoints;
- наличие тестовых данных через factories;
- возможность безопасно рефакторить backend.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/PublicAnimeApiTest.php`

Покрываемые сценарии:

1. `GET /api/v1/public/anime` возвращает список anime.
2. `GET /api/v1/public/anime?per_page=1` уважает pagination limit.
3. `GET /api/v1/public/anime/{anime}` возвращает детали anime.
4. `GET /api/v1/public/tags` возвращает список tags.
5. `GET /api/v1/public/anime/{anime}/episodes` возвращает episodes.
6. Неизвестный anime возвращает JSON `404`.

## Требования

Могут понадобиться factories:

- `AnimeFactory`
- `EpisodeFactory`
- `TagFactory`

Если factories отсутствуют, их нужно добавить.

## Проверки

После реализации:

```bash
php -l tests/Feature/Api/PublicAnimeApiTest.php
php artisan test --filter=PublicAnimeApiTest
```

## Реализация

Добавлены factories:

- `database/factories/AnimeFactory.php`
- `database/factories/TagFactory.php`
- `database/factories/EpisodeFactory.php`

Добавлен тест:

- `tests/Feature/Api/PublicAnimeApiTest.php`

Также тесты выявили проблему миграций: таблица `anime_user` использовалась последующими migrations, но не имела базовой create migration.

Добавлена migration:

- `database/migrations/2025_12_20_123222_create_anime_user_pivot_table.php`

Она создает pivot table:

- `user_id`
- `anime_id`
- `status`
- timestamps
- primary key `[user_id, anime_id]`

## Текущий статус проверок

Syntax checks прошли:

```bash
php -l database/factories/AnimeFactory.php
php -l database/factories/TagFactory.php
php -l database/factories/EpisodeFactory.php
php -l tests/Feature/Api/PublicAnimeApiTest.php
php -l database/migrations/2025_12_20_123222_create_anime_user_pivot_table.php
```

Первый запуск `PublicAnimeApiTest` выявил ошибку:

```text
SQLSTATE[HY000]: no such table: anime_user
```

После добавления migration повторный запуск ушел в timeout. Причина была в нестабильных migrations для тестового окружения.

После стабилизации migrations тест успешно проходит:

```text
PASS  Tests\Feature\Api\PublicAnimeApiTest
✓ anime list returns paginated data
✓ anime list respects per page limit
✓ anime details returns resource
✓ tags endpoint returns tags
✓ anime episodes endpoint returns episodes
✓ unknown anime returns json 404

Tests: 6 passed (24 assertions)
```

## Риски

Feature tests требуют тестовой БД. Если окружение использует production-like DB, нужно убедиться, что тесты запускаются на отдельной test database или используют транзакции/RefreshDatabase.

В Laravel feature tests рекомендуется использовать:

```php
use RefreshDatabase;
```

Но если миграции в текущем окружении тяжелые или нестабильные, тесты могут работать медленно. Это нужно учитывать в CI.
