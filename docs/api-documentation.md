# API documentation

AniYume backend uses a single API documentation system: OpenAPI annotations with L5 Swagger.

## URLs

After starting the backend:

- Swagger UI: `http://localhost:8000/docs`
- OpenAPI JSON: `http://localhost:8000/api/documentation`

The production host can be used instead of `http://localhost:8000`.

## Generate the specification

Run from `aniyume-api`:

```bash
php artisan l5-swagger:generate
```

The generated specification is stored in `storage/api-docs/`.

## Source of truth

- API routes: `aniyume-api/routes/api.php`
- OpenAPI annotations: `aniyume-api/app/Http/Controllers/Api/V1/SwaggerDocumentationController.php`
- Swagger configuration: `aniyume-api/config/l5-swagger.php`
- Admin client contract: `aniyume-admin-web/src/lib/admin-api.ts`

When an endpoint is added or changed, update its route, implementation, test, and OpenAPI annotation together.

## Verification before a demo

```bash
php artisan l5-swagger:generate
php artisan route:list
php artisan test
```

Open `/docs` and verify the main public, authentication, profile, and admin operations.

## Authentication

Protected API operations use a bearer token:

```http
Authorization: Bearer <token>
Accept: application/json
```

Use the **Authorize** button in Swagger UI to test protected operations.
