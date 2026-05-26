# Test database migration stabilization — 2026-05-08

## Цель

Стабилизировать migrations так, чтобы backend Feature tests с `RefreshDatabase` могли надежно поднимать тестовую схему БД.

## Контекст

При добавлении `PublicAnimeApiTest` тесты выявили проблему:

```text
SQLSTATE[HY000]: no such table: anime_user
```

Причина: последующие migrations пытались модифицировать таблицу `anime_user`, но базовой migration, создающей эту таблицу, не было.

Уже добавлено:

- `database/migrations/2025_12_20_123222_create_anime_user_pivot_table.php`

## Подход

1. Найти migrations с `Schema::table(...)`.
2. Проверить, что таблица создается раньше.
3. Добавить guards там, где migration может запускаться в окружении с неполной схемой.
4. Избегать destructive переписывания старых migrations без необходимости.
5. Повторить feature tests.

## Правила для стабильных migrations

Для изменения существующей таблицы:

```php
if (!Schema::hasTable('table_name')) {
    return;
}

Schema::table('table_name', function (Blueprint $table) {
    if (!Schema::hasColumn('table_name', 'column_name')) {
        $table->string('column_name')->nullable();
    }
});
```

Для удаления колонок:

```php
if (Schema::hasTable('table_name') && Schema::hasColumn('table_name', 'column_name')) {
    Schema::table('table_name', function (Blueprint $table) {
        $table->dropColumn('column_name');
    });
}
```

## Проверки

После правок:

```bash
php -l changed_migration.php
php artisan test --filter=PublicAnimeApiTest
php artisan test --filter=HealthAndRoutingTest
```

## Выполненные исправления

Добавлены/стабилизированы migrations:

- `2025_12_20_123222_create_anime_user_pivot_table.php` — создает отсутствующую pivot table `anime_user`.
- `2025_12_20_123223_create_anime_user_table.php` — добавлены guards для `anime_user`, columns и rollback.
- `2026_01_04_154747_add_watch_time_to_watch_history_table.php` — убраны рискованные index mutations, добавлены guards.
- `2026_01_01_205637_add_ban_fields_to_users_table.php` — добавлены guards и корректный rollback.
- `2026_04_12_134907_add_is_premium_to_users_table.php` — добавлены guards и корректный rollback.

## Результат

Проверки синтаксиса прошли.

`PublicAnimeApiTest` теперь проходит:

```text
Tests: 6 passed (24 assertions)
```

## Значение для диплома

Стабильные migrations важны для:

- CI pipeline;
- воспроизводимого разворачивания проекта;
- надежных feature tests;
- демонстрации инженерной зрелости проекта.
