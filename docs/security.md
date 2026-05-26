# Security notes

## Overview

Aniyume security model is based on:

- Laravel Sanctum Bearer tokens;
- role-based admin access;
- Next.js API proxy;
- CORS and API exception normalization;
- anti-scraping middleware;
- honeypot trap endpoint;
- Reverb private/presence channel authorization.

## Authentication

### Token model

Backend issues Sanctum plain text tokens on login/register.

Frontend stores token in:

```text
localStorage.userToken
```

Private API calls include:

```http
Authorization: Bearer <token>
```

### Risks

Using localStorage is simple but has XSS exposure risk.

Mitigations:

- strict Content Security Policy;
- avoid rendering untrusted HTML;
- sanitize markdown/user content;
- keep dependencies updated;
- use short-lived tokens if refresh flow is added later.

## Authorization

### User endpoints

Private routes use:

```php
auth:sanctum
```

### Admin endpoints

Admin web routes use:

```php
['auth', 'admin']
```

Admin middleware checks:

```php
$request->user()->hasRole('admin')
```

Roles are stored via:

```text
roles
role_user
```

## API proxy

Frontend does not call backend directly in most application flows.

It calls:

```text
/api/external/*
```

The proxy:

- normalizes public/private backend paths;
- attaches/normalizes Bearer token;
- adds fingerprint header when available;
- centralizes backend URL configuration.

Security benefits:

- fewer hardcoded backend URLs in frontend;
- central place for auth header normalization;
- easier logging/debugging.

Security risks:

- proxy must not blindly forward secrets;
- public/private route classification must remain explicit;
- avoid exposing internal-only backend routes through proxy.

## CORS

Backend uses Laravel CORS middleware in the API stack.

Important files:

- `bootstrap/app.php`
- `config/cors.php`

Production recommendation:

- restrict allowed origins to real frontend domains;
- avoid wildcard origins with credentials;
- keep methods/headers minimal.

## API exception format

`bootstrap/app.php` normalizes API errors:

- validation: `422`;
- not found: `404`;
- unauthenticated: `401`;
- unauthorized: `403`;
- production server errors hide internal messages.

This helps frontend handle errors consistently.

## Anti-scraping and honeypot

Backend has:

- `AntiScraperMiddleware`;
- `POST /api/v1/trap` honeypot endpoint.

Purpose:

- identify bots hitting trap endpoints;
- slow down automated scraping;
- add basic abuse protection.

For diploma defense, this can be presented as application-level abuse mitigation.

## Realtime security

Watch Party uses Reverb private/presence channels.

Frontend auth endpoint:

```text
/api/external/broadcasting/auth
```

Backend target:

```text
/api/v1/broadcasting/auth
```

Auth middleware:

```php
auth:sanctum
```

Channels:

- `presence-watch-party.{roomCode}`;
- `private-user.{userId}`.

Security requirements:

- channel authorization must verify room participation;
- private user channel must allow only the same user id;
- room host-only actions must be validated server-side.

## Content security policy

Frontend sets CSP headers in `next.config.ts`.

Current policy allows:

- self scripts/styles;
- inline/eval for framework/dev compatibility;
- media from `*.libria.fun`;
- websocket connections.

Production hardening:

- remove `unsafe-eval` if possible;
- reduce `unsafe-inline`;
- restrict `img-src` and `frame-src` to known domains;
- explicitly include production Reverb host.

## File upload security

Avatar upload validates:

- image file;
- allowed mimes;
- max size.

Recommendations:

- store uploads outside executable paths;
- serve via storage symlink/CDN;
- consider image re-encoding to strip metadata.

## Secrets

Never commit:

- `.env`;
- production keys;
- payment secrets;
- Reverb secrets;
- database credentials.

Use `.env.example` for documentation only.

## Security checklist

Before production/demo:

- [ ] verify admin users and roles;
- [ ] rotate application keys/secrets if exposed;
- [ ] set production CORS origins;
- [ ] set production Reverb host/scheme/port;
- [ ] run migrations and seed roles;
- [ ] verify queue worker;
- [ ] verify scheduler;
- [ ] review CSP;
- [ ] disable debug mode;
- [ ] verify storage permissions.
