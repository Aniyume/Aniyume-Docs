# Backend root debug files removal — 2026-05-08

## Цель

Физически удалить из корня `aniyume-backend/` одноразовые debug scripts и generated dumps, которые не являются частью production-кода.

Перед этим для таких файлов уже были добавлены правила в:

- `aniyume-backend/.gitignore`

## Удаляемые файлы

Планируемый список:

- `anilibria_docs.md`
- `anilibria_docs_utf8.md`
- `anime_list.txt`
- `anime_list_utf8.txt`
- `check_episodes.php`
- `eps_output.txt`
- `eps_utf8.txt`
- `find_anime.php`
- `find_anime_utf8.php`
- `fix_urls.php`
- `response.txt`
- `schedule.json`
- `test.php`
- `test_import.php`
- `timeout_test.txt`
- `verify_sort.php`
- `verify_urls.php`

## Почему это безопасно

- Эти файлы находятся в root backend, а не в `app/`, `routes/`, `config/`, `database/`, `resources/` или `tests/`.
- Они выглядят как временные diagnostics/import/debug scripts и output dumps.
- Reusable maintenance logic уже представлена artisan commands в `app/Console/Commands`.
- Поиск references показал только локальную связь `find_anime_utf8.php -> anime_list_utf8.txt`, что подтверждает debug/dump характер.

## Правило на будущее

Если нужна повторяемая диагностика или maintenance-задача — добавлять artisan command, а не root php script.

Примеры правильного места:

- `app/Console/Commands/CheckEpisodesCommand.php`
- `app/Console/Commands/VerifyUrlsCommand.php`

## Проверки после удаления

После удаления нужно выполнить:

- `npm run build` во frontend;
- `php artisan test` в backend;
- `php artisan schedule:list` в backend.
