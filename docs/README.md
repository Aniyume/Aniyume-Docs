# Aniyume project documentation

Этот каталог содержит техническую документацию по проекту Aniyume.

Цель документации — привести проект к уровню качественной дипломной работы: с понятной архитектурой, обоснованными решениями, проверяемыми изменениями и описанием ключевых модулей.

## Основные документы

- [`architecture-overview.md`](./architecture-overview.md) — обзор архитектуры frontend/backend.
- [`api-contract.md`](./api-contract.md) — контракт публичных и приватных API endpoints.
- [`data-model.md`](./data-model.md) — модель данных и ERD.
- [`sequences.md`](./sequences.md) — sequence diagrams для ключевых сценариев.
- [`security.md`](./security.md) — заметки по безопасности.
- [`deployment.md`](./deployment.md) — руководство по деплою.
- [`infra-docker-design-package.md`](./infra-docker-design-package.md) — целевая Docker/Compose-схема и план будущей infra-фазы.
- [`testing-strategy.md`](./testing-strategy.md) — стратегия тестирования.
- [`thesis-readiness-plan.md`](./thesis-readiness-plan.md) — план готовности к защите.
- [`demo-script.md`](./demo-script.md) — пошаговый сценарий демонстрации на защите.
- [`feature-matrix.md`](./feature-matrix.md) — матрица реализованных функций, тестов и документации.
- [`known-limitations.md`](./known-limitations.md) — известные ограничения и future work.
- [`docker-setup.md`](./docker-setup.md) — рабочий local/dev Docker Compose baseline для demo.
- [`github-migration-plan.md`](./github-migration-plan.md) — безопасный план миграции Aniyume в отдельные GitHub repositories с ветками dev/release.
- [`repository-strategy.md`](./repository-strategy.md) — целевая repository/branch/deploy стратегия для Aniyume Web, API и Admin Web.
- [`final-thesis-readiness-audit.md`](./final-thesis-readiness-audit.md) — финальный audit готовности к демонстрации и защите.
- [`user-guide.md`](./user-guide.md) — руководство пользователя.
- [`admin-guide.md`](./admin-guide.md) — руководство администратора.
- [`2026-05-08-architecture-cleanup-report.md`](./2026-05-08-architecture-cleanup-report.md) — отчет по первичной стабилизации и cleanup.
- [`2026-05-08-backend-cleanup-notes.md`](./2026-05-08-backend-cleanup-notes.md) — заметки по backend cleanup.
- [`2026-05-08-backend-root-debug-files-removal.md`](./2026-05-08-backend-root-debug-files-removal.md) — удаление временных root debug-файлов.
- [`2026-05-08-anilist-import-service-removal.md`](./2026-05-08-anilist-import-service-removal.md) — удаление legacy AniList import service.
- [`2026-05-08-unused-user-content-tables-removal.md`](./2026-05-08-unused-user-content-tables-removal.md) — удаление неиспользуемых таблиц пользовательского контента.

## Правила ведения документации

Для каждой крупной пачки изменений нужно создавать отдельный файл в `docs/`.

Документ должен содержать:

1. цель изменения;
2. список затронутых файлов/модулей;
3. почему выбран именно такой подход;
4. риски и rollback strategy, если есть;
5. команды проверки и результат.

## Быстрые проверки качества

Frontend:

```bash
cd aniyume
npm run build
```

Backend:

```bash
cd aniyume-backend
php artisan test
php artisan schedule:list
```

Для migration перед production:

```bash
php artisan migrate --pretend
```

Если команда зависит от внешней БД и зависает, нужно проверять окружение `.env`, доступность БД и миграционное состояние.
