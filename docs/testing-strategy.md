# Testing strategy

## Goal

Testing strategy defines how Aniyume should be validated before development milestones, demo, production deployment, and thesis defense.

The project should have several levels of checks:

1. syntax/static checks;
2. unit tests;
3. feature/API tests;
4. frontend build checks;
5. smoke tests;
6. manual acceptance scenarios.

## Current minimal quality gates

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
php artisan route:list
```

Changed PHP files:

```bash
php -l path/to/file.php
```

Migration safety:

```bash
php artisan migrate --pretend
```

## Recommended test pyramid

```text
Manual acceptance / demo scenarios
        ▲
Browser smoke / E2E tests
        ▲
Backend feature API tests
        ▲
Unit tests for services
        ▲
Syntax/static/build checks
```

## Backend unit tests

Good candidates:

- status mapping in import services;
- source selection in `EpisodeImportService`;
- user statistics calculations;
- watch progress aggregation;
- role checks;
- schedule/import helper logic.

Example naming:

```text
tests/Unit/AnimeStatusMappingTest.php
tests/Unit/WatchProgressTest.php
tests/Unit/UserStatisticsTest.php
```

## Backend feature/API tests

Priority endpoints:

Already added lightweight routing/health tests:

- `tests/Feature/HealthAndRoutingTest.php`
  - `GET /up` returns successful response;
  - unknown public API endpoint returns JSON `404`;
  - private `/api/v1/user` requires auth and returns JSON `401`.

Public catalog API test work started:

- `tests/Feature/Api/PublicAnimeApiTest.php`
- factories for `Anime`, `Episode`, `Tag`
- migration fix for missing `anime_user` pivot table

Current result: the first public API test run exposed a real migration consistency issue (`anime_user` was used before being created). After adding the missing migration and stabilizing several table-altering migrations, `PublicAnimeApiTest` passes with 6 tests and 24 assertions.

### Public

- `GET /api/v1/public/anime`
- `GET /api/v1/public/anime/{anime}`
- `GET /api/v1/public/tags`
- `GET /api/v1/public/schedule`

### Auth

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/user`
- `POST /api/v1/auth/logout`

Implemented:

- `tests/Feature/Api/AuthApiTest.php`
  - register returns token;
  - login returns token;
  - authenticated `/user` works;
  - unauthenticated `/user` returns `401`;
  - logout works;
  - invalid password returns validation error.

Current result:

```text
AuthApiTest — 6 passed (24 assertions)
```

Implemented for profile:

- `tests/Feature/Api/ProfileApiTest.php`
  - auth is required;
  - authenticated user can get full profile;
  - authenticated user can update profile;
  - invalid profile payload returns validation error.

Current result:

```text
ProfileApiTest — 4 passed (24 assertions)
```

### User activity

- `POST /api/v1/anime/{anime}/status`
- `GET /api/v1/my-anime-list/{status}`
- `POST /api/v1/watch-history`
- `GET /api/v1/watch-history/anime/{anime}/last-episode`
- `POST /api/v1/favorites`
- `POST /api/v1/ratings`

Implemented for anime list:

- `tests/Feature/Api/UserAnimeListApiTest.php`
  - auth is required;
  - set status;
  - get user status;
  - not listed anime returns `not_watching`;
  - filter list by status;
  - remove from list with `not_watching`;
  - update `episodes_watched`;
  - invalid status returns validation error.

Current result:

```text
UserAnimeListApiTest — 8 passed (30 assertions)
```

Implemented for watch history:

- `tests/Feature/Api/WatchHistoryApiTest.php`
  - auth is required;
  - store validates `episode_id`, `progress`, `delta_time`;
  - progress can be saved;
  - repeated save updates progress and increments watch time;
  - history can be fetched by anime;
  - last watched episode can be fetched;
  - history record can be deleted.

Current result:

```text
WatchHistoryApiTest — 7 passed (32 assertions)
```

Implemented for favorites:

- `tests/Feature/Api/FavoritesApiTest.php`
  - auth is required;
  - add anime to favorites;
  - duplicate favorite returns conflict;
  - list favorites;
  - check favorite status;
  - remove from favorites;
  - invalid anime id returns validation error.

Current result:

```text
FavoritesApiTest — 7 passed (25 assertions)
```

Implemented for ratings:

- `tests/Feature/Api/RatingsApiTest.php`
  - auth is required;
  - create rating;
  - repeated rating updates existing record;
  - get rating for anime;
  - list ratings;
  - delete own rating;
  - cannot delete another user's rating;
  - invalid payload returns validation error.

Current result:

```text
RatingsApiTest — 8 passed (26 assertions)
```

Implemented for comments:

- `tests/Feature/Api/CommentsApiTest.php`
  - public comments endpoint is accessible;
  - mutation endpoints require auth;
  - create comment;
  - list own comments;
  - update own comment;
  - delete own comment;
  - cannot update/delete another user's comment;
  - invalid payload returns validation error.

Current result:

```text
CommentsApiTest — 8 passed (30 assertions)
```

### Social/realtime REST part

- `POST /api/v1/friends/{userId}`
- `POST /api/v1/watch-party`
- `POST /api/v1/watch-party/{code}/join`
- `POST /api/v1/watch-party/{code}/message`

Implemented for Watch Party REST:

- `tests/Feature/Api/WatchPartyApiTest.php`
  - auth is required;
  - create room;
  - show active room;
  - join room;
  - send/fetch messages;
  - non-participant cannot send message;
  - host can sync player state;
  - non-host cannot sync player state;
  - participant can leave;
  - host can close;
  - non-host cannot close.

Current result:

```text
WatchPartyApiTest — 11 passed (49 assertions)
```

## Frontend tests

Current mandatory check:

```bash
npm run build
```

Recommended additions:

- component tests for core UI widgets;
- API client tests for path generation;
- smoke tests with Playwright.

Priority frontend smoke scenarios:

1. home page opens;
2. catalog loads;
3. anime details page opens;
4. login form validates input;
5. authenticated user can open profile;
6. watch party page can initialize.

## Manual acceptance scenarios

These scenarios should be used before thesis demo.

### Catalog scenario

1. Open home page.
2. Open catalog.
3. Filter by genre/status/year.
4. Search anime.
5. Open anime details.
6. Open episodes list.

### Auth/profile scenario

1. Register user.
2. Login.
3. Open profile.
4. Update profile.
5. Upload avatar.
6. Logout.

### Watch progress scenario

1. Login.
2. Open anime episode.
3. Watch several seconds.
4. Ensure watch history is saved.
5. Reload page.
6. Ensure last watched episode/progress is restored.

### Social scenario

1. Search user.
2. Send friend request.
3. Accept request from second account.
4. Verify friends list.

### Watch Party scenario

1. Login as host.
2. Create room.
3. Login as second user.
4. Join room.
5. Send chat message.
6. Play/pause/seek as host.
7. Verify guest player sync.
8. Close room.

### Admin scenario

1. Login as admin.
2. Open dashboard.
3. Open anime management.
4. Open episode management.
5. Start import for one anime.
6. Moderate comments.
7. Check audit logs.

## CI recommendation

Minimal CI pipeline:

```yaml
frontend:
  - npm ci
  - npm run build

backend:
  - composer install
  - php artisan test
```

Extended CI:

```yaml
backend:
  - php -l changed-files
  - php artisan test
  - php artisan route:list
  - php artisan schedule:list

frontend:
  - npm ci
  - npm run lint
  - npm run build
```

## Known environment issues

Some Laravel commands may hang if database connection is slow/unavailable.

If this happens:

```bash
php artisan optimize:clear
php artisan config:clear
```

Then verify:

- `.env` DB settings;
- database server availability;
- pending migrations;
- queue locks;
- long-running PHP processes.

## Definition of done

A change is considered done if:

- code is implemented;
- docs are updated;
- relevant tests/checks pass;
- risks are documented;
- rollback path is clear for destructive changes.
