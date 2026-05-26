# AnilistImportService removal — 2026-05-08

## Цель

Убрать неиспользуемый legacy import service, чтобы не держать два конкурирующих пайплайна импорта anime.

## Что проверено

Поиск references показал:

- `AnilistImportService` не внедряется через DI;
- нет artisan command, который вызывает `AnilistImportService`;
- активная команда `import:anime` использует `ShikimoriImportService`;
- AniList API всё еще используется точечно, но не через этот сервис:
  - `BannerService` получает баннеры;
  - `FillMissingDescriptionsCommand` может добирать описания.

## Решение

Удалить:

- `aniyume-backend/app/Services/AnilistImportService.php`

## Почему это безопасно

- текущий production import pipeline — `ImportAnimeCommand -> ShikimoriImportService`;
- service не используется routes/controllers/jobs/commands;
- сохранена точечная AniList-логика там, где она реально используется (`BannerService`, descriptions command);
- удаление уменьшает путаницу между `ShikimoriImportService` и `AnilistImportService`.

## Что не меняется

- `external_source = anilist` данные в базе не трогаются;
- `BannerService` не меняется;
- `FillMissingDescriptionsCommand` не меняется;
- поля anime, связанные с external source/id, не удаляются.

## Проверки

После удаления нужно выполнить:

- поиск `AnilistImportService` по backend;
- `php artisan test`;
- `php artisan schedule:list`;
- frontend `npm run build`.
