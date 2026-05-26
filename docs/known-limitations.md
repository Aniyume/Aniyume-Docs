# Known limitations and future work

## Purpose

This document lists known limitations of the current implementation and planned improvements.

For a thesis project, documented limitations are a strength: they show that the system was analyzed critically and has a realistic development roadmap.

## Runtime and infrastructure limitations

### Reverb/WebSocket depends on production infrastructure

Status: ⚠️ Environment-dependent

Watch Party realtime requires:

- Reverb server running;
- correct `NEXT_PUBLIC_REVERB_*` frontend env;
- websocket reverse proxy support;
- correct broadcast auth endpoint.

REST layer is covered by tests, but realtime delivery should be verified manually in deployment/demo environment.

Future work:

- add automated websocket/E2E tests;
- add Reverb health check;
- document exact production Nginx config for the target server.

### External video/API source availability

Status: ⚠️ External dependency

Anime metadata and video sources depend on external services.

Risks:

- provider downtime;
- rate limits;
- changed response format;
- unavailable video sources.

Future work:

- add provider fallback strategy;
- cache external responses;
- add provider health dashboard;
- add import retry/backoff policy.

### Queue worker required for background tasks

Status: ⚠️ Infrastructure requirement

Some jobs/imports require queue worker.

Future work:

- add supervisor/systemd configs to repository examples;
- add health endpoint for queue status;
- add failed jobs monitoring.

### Docker Compose is a working local/dev baseline, not a full production stack

Status: ✅ Working local baseline, ⚠️ production hardening still manual

The current Docker stack starts PostgreSQL, Redis, Laravel backend, Next.js frontend, queue worker, scheduler, Reverb and nginx. It was verified for local/demo usage and should be treated as the preferred reproducible demo entrypoint.

Known manual steps remain:

- generating/providing a real `APP_KEY` outside committed files;
- running migrations and seed/admin preparation manually;
- creating `storage:link` when uploaded/public files are needed;
- validating real external API/video provider credentials before demo;
- keeping the separate `aniyume-admin` scaffold outside Compose until its runtime ownership is finalized.

Future work:

- add a production Compose/profile or deployment manifests;
- add automated smoke checks after `docker compose up`;
- wire `aniyume-admin` into Compose only after final admin auth/session design is selected.

## Security limitations

### Token storage in localStorage

Status: ⚠️ Accepted trade-off

Frontend stores Sanctum Bearer token in `localStorage`.

Risk:

- XSS can expose token.

Mitigations already documented:

- CSP;
- avoid untrusted HTML;
- sanitize user content;
- keep dependencies updated.

Future work:

- consider httpOnly cookie auth;
- add refresh token/session rotation;
- add token expiry policy.

### Separate admin scaffold uses transitional bearer-token auth shell

Status: ⚠️ Transitional admin implementation

The main Laravel Blade-admin panel exists and is protected by backend `auth` + `admin` middleware. A separate Next.js `aniyume-admin` scaffold also exists and can call current read/write-lite admin API endpoints, but it does not yet have a complete backend-driven login/logout/session flow.

Current constraints:

- token is entered manually on `/login` and checked via `/api/v1/admin/auth/me`;
- in-memory storage is the default, optional `sessionStorage` fallback is temporary;
- no `localStorage` is used in the admin scaffold, but JS-visible token storage is still not equivalent to HttpOnly cookie auth;
- CRUD parity with legacy Blade-admin is partial.

Future work:

- implement backend-issued HttpOnly Secure SameSite cookie/session flow for admin;
- add admin logout/session invalidation;
- finish parity endpoints and UI screens before removing Blade-admin.

### CSP still allows some relaxed rules

Status: ⚠️ Framework compatibility

Some CSP directives may be relaxed for Next.js/dev compatibility.

Future work:

- harden CSP in production;
- remove `unsafe-eval` if possible;
- reduce `unsafe-inline`;
- restrict media/img/ws domains.

## Testing limitations

### Backend coverage is strong but not complete

Status: ✅ Strong baseline, 🔮 more possible

Covered:

- health/routing;
- public anime API;
- auth;
- user anime list;
- watch history;
- favorites;
- ratings;
- comments;
- profile read/update;
- Watch Party REST.

Not yet covered:

- avatar upload;
- friends endpoints;
- admin access;
- import commands;
- payment endpoint;
- Reverb websocket behavior;
- AI chat/provider/guardrail behavior.

Future work:

- expand profile tests to cover avatar upload;
- add `FriendshipApiTest`;
- add `AdminAccessTest`;
- add command tests for imports;
- add E2E tests;
- add AI endpoint and safety regression tests.

### Frontend has build checks but limited automated UI tests

Status: ⚠️ Build verified

Current frontend quality gate:

```bash
npm run build
```

Future work:

- add Playwright smoke tests;
- add component tests for API-heavy UI;
- add CI pipeline.

## Product limitations

### Payment endpoint is minimal

Status: ⚠️ Placeholder/business logic depends on provider

Premium subscription endpoint exists, but real payment provider integration may require additional implementation.

Future work:

- integrate provider webhook validation;
- add payment audit logs;
- add subscription expiration handling;
- add tests.

### Schedule quality depends on data completeness

Status: ⚠️ Data-dependent

Schedule endpoint can only be accurate if imported anime have correct airing metadata.

Future work:

- improve import mapping for airing days;
- add admin correction UI;
- add schedule tests.

### Admin panel test coverage is limited

Status: ⚠️ Manual/admin layer

Admin panel exists and routes are protected, but automated tests focus mostly on API.

Future work:

- add admin access tests;
- add import management tests;
- add comment moderation tests.

### AI provider availability and safety verification are environment-dependent

Status: ✅ Implemented module, ⚠️ demo depends on env/provider

The backend contains AI chat/session code, provider gateway/fallback, role policies, throttling and safety guardrails. For demo, the exact behavior depends on configured provider, API key, model and network availability.

Future work:

- document a deterministic demo prompt set;
- add tests for AI role policies, tool allowlist and guardrail refusals;
- expose a simple health/status check that does not leak provider secrets.

## Database limitations

### Destructive cleanup migrations require backup

Status: ⚠️ Operational risk

Unused user content tables were removed via migration.

Rollback restores structure but not deleted data.

Future work:

- production backup checklist;
- migration dry-run procedure;
- data retention policy.

## Documentation limitations

Status: ✅ Strong baseline

Already documented:

- architecture;
- API contracts;
- ERD/data model;
- sequence diagrams;
- security;
- deployment;
- testing;
- docker setup;
- demo script;
- feature matrix.

Future work:

- add screenshots for thesis text;
- export ERD diagram as image;
- add final user manual;
- add admin manual.

## Future roadmap

Recommended next development milestones:

1. Friend system automated tests.
2. Profile/avatar automated tests.
3. Admin access and moderation tests.
4. Frontend Playwright smoke tests.
5. CI pipeline.
6. Production monitoring/health checks.
7. Docker Compose or deployment automation.
8. Payment provider integration hardening.
9. WebSocket E2E tests.

## Thesis framing

These limitations do not make the project incomplete. They define future development directions and demonstrate that the system has been evaluated like a real software product.
