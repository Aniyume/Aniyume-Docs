# Comments API tests — 2026-05-08

## Цель

Покрыть тестами комментарии к anime.

Comments — важная community-функция проекта:

- пользователи оставляют комментарии;
- публичная страница anime показывает комментарии;
- пользователь видит свои комментарии;
- автор может редактировать и удалять свой комментарий;
- чужие комментарии должны быть защищены от изменения.

## Планируемые тесты

Файл:

- `aniyume-backend/tests/Feature/Api/CommentsApiTest.php`

Сценарии:

1. Public comments endpoint доступен без auth.
2. Private comment mutation endpoints требуют auth.
3. Пользователь может создать comment.
4. Пользователь может получить свои comments.
5. Пользователь может обновить свой comment.
6. Пользователь может удалить свой comment.
7. Пользователь не может обновить/удалить чужой comment.
8. Невалидный payload возвращает validation error.

## Проверки

```bash
php -l tests/Feature/Api/CommentsApiTest.php
php artisan test --filter=CommentsApiTest
```

## Результат

Comments tests добавлены и проходят:

```text
PASS  Tests\Feature\Api\CommentsApiTest
✓ public comments endpoint is accessible
✓ comment mutation endpoints require authentication
✓ user can create comment
✓ user can get own comments
✓ user can update own comment
✓ user can delete own comment
✓ user cannot update or delete another users comment
✓ invalid comment payload returns validation error

Tests: 8 passed (30 assertions)
```

Дополнительно повторно проверены:

```text
RatingsApiTest — 8 passed (26 assertions)
FavoritesApiTest — 7 passed (25 assertions)
WatchHistoryApiTest — 7 passed (32 assertions)
```

## Значение для диплома

Comments tests вместе с ratings подтверждают, что проект имеет полноценный community-layer, а не только каталог и просмотр.
