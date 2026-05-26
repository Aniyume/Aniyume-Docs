# Final smoke checklist before thesis defense

## Purpose

This checklist is a compact day-of-defense runbook for the Aniyume demo. It is intentionally practical: verify only what must work for a confident live demonstration, prepare backup paths, and avoid risky unrehearsed actions.

Scope: documentation/checklist only. No architecture, runtime code, or source behavior is changed by this document.

## Golden rule for the demo

Show the strongest and most stable path first:

1. Docker/local stack health.
2. Public catalog and anime details.
3. Authenticated user flows.
4. Community features.
5. Watch Party, only after Reverb is verified.
6. Operational admin via legacy Laravel admin.
7. Admin API / Next admin scaffold as migration-readiness.
8. AI module, only with verified provider/fallback.
9. Tests and documentation pack.

Do not improvise with unverified external providers, secrets, destructive admin actions, or unrehearsed migrations during the defense.

## Minimal commands before defense

Run from repository root unless noted otherwise.

```powershell
docker compose config
docker compose up --build -d
docker compose ps
curl.exe --max-time 60 http://localhost:8088/nginx-health
curl.exe --max-time 60 http://localhost:8000/up
```

Backend test baseline, if local PHP environment is ready:

```powershell
cd aniyume-backend
php artisan test
php artisan route:list
php artisan schedule:list
```

Frontend build baseline, if local Node environment is ready:

```powershell
cd aniyume
npm run build
```

Optional standalone admin scaffold, only if rehearsed:

```powershell
cd aniyume-admin
npm run dev
```

## Must check before defense

### 1. Docker and service health

- [ ] `docker compose config` succeeds.
- [ ] `docker compose up --build -d` completes without failed services.
- [ ] `docker compose ps` shows the expected stack running/healthy:
  - [ ] `db`;
  - [ ] `redis`;
  - [ ] `backend`;
  - [ ] `frontend`;
  - [ ] `nginx`;
  - [ ] `queue-worker`;
  - [ ] `scheduler`;
  - [ ] `reverb`.
- [ ] `http://localhost:8088/nginx-health` responds.
- [ ] `http://localhost:8000/up` responds.
- [ ] Main demo entrypoint opens: `http://localhost:8088`.
- [ ] No terminal window exposes real `.env` secrets/API keys on screen.

### 2. Demo accounts and data

- [ ] Regular user #1 can log in.
- [ ] Regular user #2 can log in in a second browser/session for Watch Party.
- [ ] Admin user can log in.
- [ ] Demo anime records exist.
- [ ] Tags/genres exist.
- [ ] At least one anime has episodes.
- [ ] At least one anime has a usable player/source path or a safe backup explanation.
- [ ] There is safe demo data for favorites/list/history/rating/comment flows.
- [ ] Import logs or import dashboard data exist if admin imports are shown.

### 3. Frontend smoke path

- [ ] Home page opens through `http://localhost:8088`.
- [ ] Catalog/list page loads.
- [ ] Search works with a known query.
- [ ] Filters work with a known safe filter, for example genre/status/year.
- [ ] Anime details page opens.
- [ ] Episodes list is visible for a prepared anime.
- [ ] Player route/page opens or fallback source explanation is ready.
- [ ] Login page opens and authenticates user #1.
- [ ] Profile/user area opens after login.

### 4. Auth and user flows

- [ ] Login works for regular user #1.
- [ ] Authenticated API-dependent pages do not redirect unexpectedly.
- [ ] Add anime to `watching` or `planned` list.
- [ ] User list/profile reflects the change.
- [ ] Remove or change the list status only if this was rehearsed.
- [ ] Add favorite.
- [ ] Add/update rating.
- [ ] Add comment with safe text.
- [ ] Comments list displays the new or existing comment.
- [ ] Watch history/progress is shown or backend behavior is ready to explain.

### 5. Watch Party

- [ ] Reverb service is running before showing realtime.
- [ ] User #1 can create a Watch Party room.
- [ ] User #2 can join via code/link in a separate browser/session.
- [ ] Chat message is delivered.
- [ ] Host play/pause/seek sync works.
- [ ] Host-only behavior is ready to explain as backend-validated.
- [ ] Backup path is ready if realtime is unstable: REST tests/docs/sequence diagram.

### 6. Admin

- [ ] Legacy Laravel admin under `/admin/*` opens.
- [ ] Admin login works.
- [ ] Dashboard opens.
- [ ] Anime management page opens.
- [ ] Tags/genres management page opens, if shown.
- [ ] Episodes/import page opens, if shown.
- [ ] Comments/users/audit logs open, if shown.
- [ ] Avoid destructive actions unless a throwaway demo record is prepared.
- [ ] Correct wording is rehearsed: legacy Blade-admin is operational; Next admin scaffold and admin API show migration-readiness.

### 7. Admin API and imports

- [ ] Admin API surface is ready to mention under `/api/v1/admin/*`.
- [ ] If showing API manually, use a prepared token/session and non-secret request history.
- [ ] `GET /api/v1/admin/auth/me` is known to work with prepared auth.
- [ ] `GET /api/v1/admin/dashboard` is known to work with prepared auth.
- [ ] `GET /api/v1/admin/imports/dashboard` or logs are known to work if imports are discussed.
- [ ] Do not run live destructive import/update/delete actions unless rehearsed and safe.

### 8. AI module

- [ ] AI provider/env/fallback behavior is verified before the demo.
- [ ] User auth/token for AI is ready.
- [ ] Safe prompts are prepared and do not require secrets or external private data.
- [ ] `POST /api/v1/ai/chat` behavior is known in the current environment.
- [ ] AI session/history behavior is ready to mention.
- [ ] Provider gateway/fallback, throttling, role/policy, tool allowlist, and guardrails are ready to explain.
- [ ] Backup path is ready if provider/network fails: show endpoint/docs/module structure and explain external provider dependency.

## Nice to check before defense

- [ ] `php artisan test` result is fresh and known.
- [ ] `npm run build` result is fresh and known.
- [ ] `php artisan schedule:list` is ready to show for operational maturity.
- [ ] `php artisan route:list` is ready to show for API/admin/AI route visibility.
- [ ] `docs/demo-script.md` is open or bookmarked.
- [ ] `docs/final-thesis-readiness-audit.md` is open or bookmarked.
- [ ] `docs/feature-matrix.md`, `docs/testing-strategy.md`, and `docs/known-limitations.md` are easy to open.
- [ ] Browser tabs are prearranged in the planned order.
- [ ] Two browser sessions are prepared for Watch Party.
- [ ] Screen zoom/font size is readable for the audience.
- [ ] Network-dependent features are tested on the exact defense network, if possible.

## Recommended manual demonstration order

### Step 0. Start with health, not code

Show briefly:

- `docker compose ps`;
- `http://localhost:8088/nginx-health`;
- `http://localhost:8000/up`.

Message: the project runs as a full local demo stack: frontend, backend, database, Redis, Reverb, worker/scheduler, nginx.

### Step 1. Public product value

Show:

1. Home/catalog.
2. Search.
3. Filters.
4. Anime details.
5. Episodes/player route.

Why first: this is visually clear and demonstrates the core user-facing product before technical details.

### Step 2. Authenticated user value

Show:

1. Login.
2. Profile/user area.
3. Add anime to list.
4. Favorite.
5. Rating.
6. Comment.
7. Watch history/progress if stable in UI.

Why second: this proves the platform is not just a public catalog but has personalized user state.

### Step 3. Realtime Watch Party

Show only if already verified in the same environment:

1. User #1 creates room.
2. User #2 joins.
3. Chat message.
4. Play/pause/seek sync.
5. Host-only control explanation.

Why third: it is technically impressive, but websocket demos are environment-sensitive.

### Step 4. Admin operational layer

Show:

1. `/admin/*` login/dashboard.
2. Anime management.
3. Episodes/imports.
4. Comments/users/audit logs if stable.

Message: operational admin exists today in Laravel Blade-admin.

### Step 5. Admin API and Next admin scaffold

Show lightly:

1. Mention `/api/v1/admin/*` endpoints.
2. Optionally show Next admin scaffold only if it starts cleanly and auth is prepared.

Message: standalone Next admin migration is in progress; admin API/scaffold demonstrate readiness, not complete replacement.

### Step 6. AI module

Show if provider/fallback was tested:

1. Authenticated AI request or UI, if available.
2. Safe prompt, for example: `Recommend anime for a user who likes dark fantasy and short series.`
3. Mention backend-only provider secret handling.
4. Mention sessions, gateway/fallback, throttling, role policy, tool allowlist, guardrails.

If not stable, do not force a live request. Show docs/routes/module summary instead.

### Step 7. Engineering maturity

Show:

1. Test result: `php artisan test`.
2. Build result: `npm run build`, if clean.
3. Documentation pack in `docs/`.
4. Known limitations.

Finish with a concise summary: fullstack platform, user flows, realtime layer, admin/admin API, AI backend module, Docker stack, tests and documentation.

## Backup steps if something breaks

### Docker stack does not fully start

1. Run `docker compose ps` and identify failed service.
2. Check that `.env`/Docker env and `APP_KEY` are prepared.
3. Check DB health and migrations.
4. If nginx is down but backend/frontend are up, use direct service URLs only if rehearsed.
5. If Docker cannot be recovered quickly, switch to screenshots/docs/test output rather than debugging live.

### Backend health fails

1. Verify `http://localhost:8000/up`.
2. Check migrations and DB connection.
3. Avoid live database repair during defense.
4. Show documented API/test coverage and recorded known-good command output if available.

### Frontend fails or page crashes

1. Refresh once.
2. Use another prepared route/tab.
3. Continue with backend health/API/admin/docs.
4. Do not open dev tools and debug source code unless explicitly asked.

### Auth fails

1. Try the prepared backup user.
2. Use an already authenticated browser session if available.
3. Show public flows and explain Sanctum-protected endpoints.
4. Do not reset passwords or expose credentials on screen.

### Watch Party/Reverb fails

1. Do not spend more than 1-2 minutes retrying.
2. Show room lifecycle concept via REST/API tests.
3. Open `docs/sequences.md` and/or Watch Party test notes.
4. Explain that websocket delivery depends on Reverb/reverse proxy runtime, while backend REST foundation is implemented and tested.

### External video/source fails

1. Continue with catalog/details/user flows.
2. Show that player source endpoint/data exists.
3. Explain dependency on external provider/source availability.

### Admin scaffold fails

1. Fall back to legacy Laravel Blade-admin.
2. Explain Next admin scaffold as migration-readiness.
3. Mention admin API subset rather than forcing the standalone UI.

### Imports fail

1. Do not run repeated live imports.
2. Show import dashboard/logs if available.
3. Explain queue/scheduler/import pipeline.
4. Mention external provider/rate-limit dependency.

### AI provider fails

1. Do not expose or edit API keys live.
2. Show AI endpoint/session routes and docs.
3. Explain provider gateway/fallback, role policy, throttling, guardrails, and tool allowlist.
4. State clearly that provider availability depends on runtime env/API key/network.

## Critical backup-plan areas

These features are valuable but should have fallback content ready:

| Area | Risk | Primary demo | Backup |
|---|---|---|---|
| Docker | Env, migrations, service startup | `docker compose ps` + health URLs | Use pre-captured results/docs; avoid live debugging |
| Reverb/Watch Party | Websocket/proxy/browser session instability | Two-user room + chat + sync | REST tests, sequence diagram, explanation |
| AI | External provider/API key/network | Safe authenticated prompt | Routes/docs/module structure/fallback explanation |
| External video/import providers | Rate limits/source downtime | Prepared anime/source/import logs | Catalog/details + endpoint/log explanation |
| Next admin scaffold | Transitional auth/session flow | Optional scaffold demo | Legacy Blade-admin + admin API wording |
| Admin destructive actions | Accidental data changes | Open pages/read-only review | Prepared throwaway record or skip mutation |
| Auth | Expired token/wrong demo password | Prepared user/admin accounts | Already-authenticated session/backup account |

## What to verify specifically

### AI

- Auth is required and works for the prepared user.
- A safe prompt returns either a valid answer or expected fallback.
- Session/history behavior is understandable.
- No provider secret is visible in frontend or terminal.
- Safety positioning is clear: backend gateway, provider fallback, role/policy, throttling, tool allowlist, output guardrails.

### Admin

- `/admin/*` is the reliable operational admin path.
- Admin user can access dashboard.
- Anime/tags/episodes/import pages open.
- Comments/users/audit logs open only if stable.
- `/api/v1/admin/*` is positioned as API foundation for standalone admin.
- `aniyume-admin` is described as scaffold/transitional, not a complete replacement.

### Docker

- Compose config is valid.
- Core services are healthy/running.
- Nginx health responds.
- Backend health responds.
- Reverb is running before Watch Party.
- Queue worker/scheduler are running before discussing imports/operations.

### Imports/admin routes

- Admin import dashboard/log route opens if shown.
- Import logs exist or backup explanation is ready.
- Do not trigger a long or external-provider-sensitive import during the main demo unless already rehearsed.
- Admin API routes are discussed with prepared auth only.

## What not to show if risky

- Real `.env`, API keys, provider tokens, passwords, or secret-bearing terminal output.
- Live editing of configuration during defense.
- Fresh migrations/resets on the demo database unless specifically rehearsed and disposable.
- Destructive admin delete/update actions on important demo data.
- Unrehearsed import runs against external providers.
- Unverified AI prompts that can produce irrelevant, unsafe, or slow responses.
- Long terminal debugging sessions.
- Claims that Next admin fully replaces legacy admin.
- Claims that Docker setup is production-hardened; present it as local/demo baseline.

## Final five-minute pre-show checklist

- [ ] Docker services are up and health checks pass.
- [ ] Main site is open at `http://localhost:8088`.
- [ ] User #1, user #2, and admin sessions are ready.
- [ ] Prepared anime/details/player page is bookmarked.
- [ ] Watch Party was tested or backup docs are open.
- [ ] AI prompt was tested or backup docs are open.
- [ ] Admin panel is open or bookmarked.
- [ ] Test/build results are known.
- [ ] Demo script and final audit are open.
- [ ] No secrets are visible.
