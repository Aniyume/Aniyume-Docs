# Auth API feature tests — 2026-05-08

## Цель

Добавить feature tests для авторизации, потому что auth — критический слой платформы.

Для дипломного уровня важно показать, что система проверяет:

- регистрацию;
- вход;
- доступ к текущему пользователю;
- logout;
- запрет private endpoints без token.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/AuthApiTest.php`

Сценарии:

1. Пользователь может зарегистрироваться и получить token.
2. Пользователь может войти по email/password и получить token.
3. Authenticated user может получить `/api/v1/user`.
4. Unauthenticated request к `/api/v1/user` возвращает `401`.
5. Authenticated user может выполнить logout.
6. Login с неверным паролем возвращает ошибку.

## Проверки

```bash
php -l tests/Feature/Api/AuthApiTest.php
php artisan test --filter=AuthApiTest
```

## Результат

Auth API tests добавлены и успешно проходят:

```text
PASS  Tests\Feature\Api\AuthApiTest
✓ user can register and receive token
✓ user can login and receive token
✓ authenticated user can fetch current user
✓ current user endpoint requires authentication
✓ authenticated user can logout
✓ login with invalid password returns validation error

Tests: 6 passed (24 assertions)
```

Дополнительно после добавления Auth tests повторно проверены:

```text
PublicAnimeApiTest — 6 passed (24 assertions)
HealthAndRoutingTest — 3 passed (5 assertions)
```

## Значение для проекта

Auth tests усиливают:

- надежность пользовательских сценариев;
- безопасность private API;
- уверенность перед добавлением tests для favorites/watch-history/watch-party.
