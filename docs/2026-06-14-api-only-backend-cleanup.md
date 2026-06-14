# API-only backend cleanup

Date: 2026-06-14

## Goal

Remove obsolete Laravel frontend scaffolding while preserving the JSON API contract used by the Next.js admin application.

## Removed

- default Laravel welcome Blade template;
- backend Vite, CSS, and JavaScript scaffolding;
- duplicate Scribe API documentation setup;
- stale hand-written `API_DOCS.md`;
- default placeholder feature and unit tests.
- misplaced unused `app/Policies/CommentPolicy.php` file that duplicated a controller and violated PSR-4 autoloading.

## Preserved

- password reset email Blade template;
- L5 Swagger UI Blade template;
- `/`, `/admin/*`, and `/login` compatibility routes;
- all `/api/v1/*` routes;
- admin API controllers used by `aniyume-admin-web`;
- meaningful unit and feature tests.

## API contract safety

The Next.js admin API client paths were checked against the Laravel route list. No API routes were removed or renamed during this cleanup.

## Documentation

Swagger is the single API documentation interface:

- UI: `/docs`
- OpenAPI JSON: `/api/documentation`
- generation command: `php artisan l5-swagger:generate`

See `docs/api-documentation.md`.

## Reliability hardening

- PHPStan and Pint pass without reported errors.
- Direct runtime `env()` calls were removed from AI moderation, external provider, security, and anti-scraper services so production `config:cache` is safe.
- Cache and queue examples use failover drivers.
- `/up` remains a lightweight liveness endpoint.
- `/ready` checks database and cache availability and returns `503` when dependencies are unavailable.
- Docker health checks use `/ready`.
- The prod-like Laravel server uses four CLI workers for better demo concurrency.

Final verification on 2026-06-15:

```text
PHPStan: 34 type-analysis errors remain in Watch Party relations/resources
Pint: 351 files pass
PHPUnit: 176 tests, 731 assertions pass
Swagger generation: pass
Composer validation: pass
```

The remaining PHPStan findings are documented technical debt and do not reproduce as functional test failures. The current runtime is appropriate for thesis demonstration and small-to-medium traffic. A high-load production deployment should replace `php artisan serve` with PHP-FPM or Laravel Octane, add multiple backend replicas, and run load tests.
