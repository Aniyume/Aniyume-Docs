# Profile API tests — 2026-05-08

## Цель

Покрыть тестами API профиля пользователя.

Profile API важен для личного кабинета:

- получение полного профиля;
- обновление имени, bio и custom status;
- валидация payload;
- запрет доступа без auth.

## Реализация

Файл:

- `aniyume-backend/tests/Feature/Api/ProfileApiTest.php`

Сценарии:

1. Profile endpoints требуют auth.
2. Authenticated user может получить full profile.
3. Authenticated user может обновить profile fields.
4. Невалидный payload возвращает validation error.

## Проверки

```bash
php -l tests/Feature/Api/ProfileApiTest.php
php artisan test --filter=ProfileApiTest
```

## Значение для диплома

Profile tests закрывают личный кабинет пользователя и дополняют уже покрытые auth/user activity/community сценарии.
