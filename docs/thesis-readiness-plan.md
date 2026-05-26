# Thesis readiness plan

## Goal

Bring Aniyume to the level of a strong diploma project by improving:

- architecture clarity;
- feature completeness;
- code quality;
- documentation;
- testing;
- deployment readiness;
- demo reliability.

## Positioning

Aniyume can be presented as:

> A fullstack anime streaming and community platform with catalog management, watch progress tracking, social features, realtime watch parties, admin panel, background imports, and external API integrations.

## Strong thesis points

### 1. Fullstack architecture

- Next.js frontend;
- Laravel backend;
- REST API;
- WebSocket realtime layer;
- DB-backed scheduler/queues.

### 2. Real user workflows

- registration/login;
- catalog browsing;
- viewing episodes;
- favorites and watch lists;
- watch history;
- comments and ratings;
- friends;
- watch parties.

### 3. Admin and operational layer

- admin panel;
- import management;
- user management;
- comment moderation;
- audit logs;
- scheduled imports.

### 4. External integrations

- Shikimori import;
- AniLibria/Kodik/VideoCDN episode sources;
- AniList banners/descriptions;
- Reverb realtime server.

### 5. Engineering practices

- documented architecture;
- API contracts;
- ERD;
- sequence diagrams;
- security notes;
- deployment guide;
- testing strategy;
- cleanup reports.

## Required documentation for defense

Already created:

- `architecture-overview.md`
- `api-contract.md`
- `data-model.md`
- `sequences.md`
- `security.md`
- `deployment.md`
- `testing-strategy.md`

Recommended next docs:

- `demo-script.md` — step-by-step defense demo scenario — added;
- `feature-matrix.md` — list of features and implementation status — added;
- `known-limitations.md` — limitations and future work — added;
- `user-guide.md` — user-facing guide — added;
- `admin-guide.md` — admin panel guide — added;
- `performance-notes.md` — caching/import/video/realtime performance;
- `admin-guide.md` — admin panel usage.

## Defense demo script outline

1. Open home page and catalog.
2. Show filtering/search.
3. Open anime page.
4. Show episodes/player.
5. Login as user.
6. Add anime to watching/favorites.
7. Watch part of episode and show progress/history.
8. Add rating/comment.
9. Show friends flow.
10. Create Watch Party.
11. Join from another account/browser.
12. Send chat message.
13. Sync play/pause/seek.
14. Login as admin.
15. Show admin dashboard, anime/episode management, comment moderation.
16. Show scheduler/import commands and docs.

## Code quality roadmap

### Done

- fixed frontend/backend API proxy;
- fixed frontend build;
- cleaned legacy routes/controllers;
- cleaned unused models/services/root debug files;
- added migration for unused tables;
- documented architecture and contracts.

### Next

- expand feature tests beyond public API and auth;
- user anime list feature tests — passing;
- watch-history feature tests — passing;
- favorites feature tests — passing;
- ratings feature tests — passing;
- comments feature tests — passing;
- Watch Party REST tests — passing;
- profile feature tests — passing;
- add frontend smoke/E2E tests;
- improve API response consistency;
- add production-ready logging/monitoring notes.

## Testing roadmap

Minimum before defense:

- `npm run build` passes;
- `php artisan test` passes;
- public catalog endpoint works;
- auth flow works manually;
- watch party works manually;
- admin login works;
- scheduler list is correct.

Better before defense:

- public API feature tests — started and passing;
- auth feature tests — passing;
- user anime list feature tests — passing;
- watch history tests — passing;
- favorites tests — passing;
- ratings tests — passing;
- comments tests — passing;
- Watch Party REST tests — passing;
- smoke test script;
- seed/demo data.

## Deployment readiness roadmap

Minimum:

- `.env.example` for frontend/backend;
- deployment guide;
- scheduler configured;
- queue worker configured;
- Reverb configured;
- storage linked.

Better:

- Docker Compose;
- CI pipeline;
- database backup strategy;
- monitoring/logging;
- automated smoke checks.

## Known risks to address

- localStorage token XSS risk;
- production Reverb configuration;
- external video source availability;
- import API rate limits;
- DB migration safety for destructive cleanup;
- limited automated test coverage.

## Final thesis checklist

- [ ] project builds successfully;
- [ ] backend tests pass;
- [ ] migrations are applied;
- [ ] demo data is prepared;
- [ ] admin account is ready;
- [ ] two user accounts are ready for Watch Party demo;
- [ ] Reverb is running;
- [ ] queue worker is running;
- [ ] scheduler is configured;
- [ ] docs are up to date;
- [ ] demo script is rehearsed;
- [ ] known limitations are documented.
