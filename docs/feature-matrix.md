# Feature matrix

## Legend

| Status | Meaning |
|---|---|
| ✅ Done | Реализовано и готово к демонстрации |
| 🧪 Tested | Покрыто automated tests |
| 📚 Documented | Описано в docs |
| ⚠️ Partial | Работает частично или зависит от окружения |
| 🔮 Future | Запланировано как развитие |

## Frontend

| Feature | Status | Notes |
|---|---:|---|
| Home page | ✅ | Static/public entry |
| Catalog/list | ✅ 📚 | Uses public anime API |
| Search/filter | ✅ 📚 | Backend supports query filters |
| Anime details page | ✅ 📚 | Details, banner, episodes |
| Video player | ✅ | P2P player is client-only to avoid SSR build issues |
| Bookmarks/profile navigation | ✅ | Integrated with backend status/favorites APIs |
| Schedule page | ✅ | Uses backend schedule endpoint |
| Watch Party page | ✅ ⚠️ | REST tested; realtime depends on Reverb runtime |
| AI chat UI/API integration | ✅ ⚠️ 📚 | Backend AI module exists; provider/API-key availability is environment-dependent |

## Backend public API

| Feature | Status | Tests |
|---|---:|---|
| Public anime list | ✅ 🧪 📚 | `PublicAnimeApiTest` |
| Public anime details | ✅ 🧪 📚 | `PublicAnimeApiTest` |
| Public tags | ✅ 🧪 📚 | `PublicAnimeApiTest` |
| Public episodes | ✅ 🧪 📚 | `PublicAnimeApiTest` |
| Public comments | ✅ 🧪 📚 | `CommentsApiTest` |
| Public schedule | ✅ 📚 | Manual/API docs |
| AI chat | ✅ ⚠️ 📚 | `POST /api/v1/ai/chat`; safety/provider runtime depends on env/config |
| AI chat sessions | ✅ ⚠️ 📚 | `GET /api/v1/ai/chat/sessions`, `GET /api/v1/ai/chat/sessions/{sessionId}` |

## Auth and profile

| Feature | Status | Tests |
|---|---:|---|
| Register | ✅ 🧪 📚 | `AuthApiTest` |
| Login | ✅ 🧪 📚 | `AuthApiTest` |
| Current user `/user` | ✅ 🧪 📚 | `AuthApiTest` |
| Logout | ✅ 🧪 📚 | `AuthApiTest` |
| Profile read/update | ✅ 🧪 📚 | `ProfileApiTest` |
| Avatar upload | ✅ 📚 | Future feature tests recommended |

## User activity

| Feature | Status | Tests |
|---|---:|---|
| Anime status/list | ✅ 🧪 📚 | `UserAnimeListApiTest` |
| Episodes watched | ✅ 🧪 📚 | `UserAnimeListApiTest` |
| Watch history store | ✅ 🧪 📚 | `WatchHistoryApiTest` |
| Watch history by anime | ✅ 🧪 📚 | `WatchHistoryApiTest` |
| Last watched episode | ✅ 🧪 📚 | `WatchHistoryApiTest` |
| Favorites | ✅ 🧪 📚 | `FavoritesApiTest` |
| Ratings | ✅ 🧪 📚 | `RatingsApiTest` |
| Comments | ✅ 🧪 📚 | `CommentsApiTest` |

## Social and realtime

| Feature | Status | Tests |
|---|---:|---|
| Friends REST endpoints | ✅ 📚 | Future feature tests recommended |
| User search | ✅ 📚 | Future feature tests recommended |
| Watch Party room create/show | ✅ 🧪 📚 | `WatchPartyApiTest` |
| Watch Party join/leave | ✅ 🧪 📚 | `WatchPartyApiTest` |
| Watch Party messages | ✅ 🧪 📚 | `WatchPartyApiTest` |
| Watch Party player sync REST | ✅ 🧪 📚 | `WatchPartyApiTest` |
| Watch Party realtime events | ✅ ⚠️ 📚 | Requires Reverb runtime/manual demo |
| Friend invites realtime | ✅ ⚠️ 📚 | Requires Reverb runtime/manual demo |

## Admin panel

| Feature | Status | Notes |
|---|---:|---|
| Admin auth | ✅ 📚 | Protected by `auth` + `admin` middleware |
| Dashboard | ✅ | Web admin layer |
| Anime management | ✅ | Admin controllers/routes |
| Episode management | ✅ | Includes import job fixes |
| Tag management | ✅ | Admin controllers/routes |
| User management | ✅ | Admin controllers/routes |
| Comment moderation | ✅ | Admin controllers/routes |
| Audit logs | ✅ | Operational/admin visibility |

## Admin API and separate admin scaffold

| Feature | Status | Notes |
|---|---:|---|
| Admin API auth/me | ✅ 📚 | `GET /api/v1/admin/auth/me`, protected by `auth:sanctum` + `admin` |
| Admin API dashboard | ✅ 📚 | `GET /api/v1/admin/dashboard` |
| Admin API anime management | ✅ ⚠️ 📚 | list/create/update/delete exist; endpoint set is smaller than full Blade-admin parity |
| Admin API tag write operations | ✅ ⚠️ 📚 | create/update/delete exist; read parity should be checked through current admin UI/API behavior |
| Admin API import dashboard/logs/run | ✅ ⚠️ 📚 | read/run endpoints exist; operational success depends on external providers and queue/import setup |
| Next.js `aniyume-admin` scaffold | ✅ ⚠️ 📚 | Pages for login/dashboard/anime/imports exist; auth flow is transitional bearer-token shell, not final cookie/session auth |

## Background jobs and scheduler

| Feature | Status | Notes |
|---|---:|---|
| Anime import | ✅ 📚 | `import:anime` |
| Episode import | ✅ 📚 | `import:episodes` canonical command |
| Ongoing sync | ✅ 📚 | `episodes:sync-ongoing --days=1` |
| Scheduler config | ✅ 📚 | `routes/console.php` |
| Queue worker | ⚠️ 📚 | Required in production for queued jobs |

## AI module

| Feature | Status | Notes |
|---|---:|---|
| AI chat endpoint | ✅ ⚠️ 📚 | `POST /api/v1/ai/chat` behind `auth:sanctum` and AI throttle |
| AI session history | ✅ ⚠️ 📚 | Session list/detail endpoints exist |
| Provider gateway/fallback | ✅ ⚠️ 📚 | DeepSeek/stub behavior depends on `config/ai.php` and secrets in env |
| Safety guardrails | ✅ 📚 | Output guard, tool capability gate, role/policy config are present |
| AI tools | ✅ ⚠️ 📚 | Whitelisted tools exist for user/admin summaries/search; scope depends on role policy |

## Documentation

| Document | Status |
|---|---:|
| Architecture overview | ✅ |
| API contract | ✅ |
| Data model / ERD | ✅ |
| Sequence diagrams | ✅ |
| Security notes | ✅ |
| Deployment guide | ✅ |
| Testing strategy | ✅ |
| Thesis readiness plan | ✅ |
| Demo script | ✅ |
| Feature matrix | ✅ |
| Known limitations | ✅ |
| Docker setup guide | ✅ |
| Final thesis readiness audit | ✅ |

## Automated test summary

Current meaningful backend feature tests:

| Test class | Tests | Assertions |
|---|---:|---:|
| `HealthAndRoutingTest` | 3 | 5 |
| `PublicAnimeApiTest` | 6 | 24 |
| `AuthApiTest` | 6 | 24 |
| `UserAnimeListApiTest` | 8 | 30 |
| `WatchHistoryApiTest` | 7 | 32 |
| `FavoritesApiTest` | 7 | 25 |
| `ProfileApiTest` | 4 | 24 |
| `RatingsApiTest` | 8 | 26 |
| `CommentsApiTest` | 8 | 30 |
| `WatchPartyApiTest` | 11 | 49 |

Total:

```text
70 tests
272 assertions
```

## Recommended future tests

- `FriendshipApiTest`
- `AdminAccessTest`
- `ImportCommandTest`
- frontend Playwright smoke tests
