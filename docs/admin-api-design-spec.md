# Admin API design spec

## Назначение

Документ фиксирует целевой JSON contract для переноса текущей Laravel Blade-admin панели в отдельный Next.js admin client. Это **design/spec package**, а не инструкция по немедленному внедрению: PHP-код, routes и контроллеры на этой фазе не меняются.

Источник функциональных требований — текущие Blade admin routes/controllers/views:

- `aniyume-backend/routes/web.php`
- `aniyume-backend/app/Http/Controllers/Admin/*`
- `aniyume-backend/resources/views/admin/*`

## Общие правила API

### Base prefix

```text
/api/v1/admin
```

### Transport

Все endpoints принимают и возвращают JSON.

Обязательные headers для защищенных endpoints:

```http
Accept: application/json
Content-Type: application/json
Authorization: Bearer <admin-token>
```

Если будет выбран cookie/Sanctum SPA режим, `Authorization` заменяется на cookie-based session contract, но JSON schema endpoints должна остаться такой же.

### Auth and roles

Все endpoints, кроме `POST /api/v1/admin/auth/login`, требуют:

- authenticated user;
- role `admin` через существующую проверку `user->hasRole('admin')`;
- JSON response при отказе.

Стандартные ошибки:

```json
{ "message": "Unauthenticated." }
```

```json
{ "message": "Access denied" }
```

```json
{
  "message": "The given data was invalid.",
  "errors": {
    "field": ["Validation message"]
  }
}
```

### Response envelope

Single resource:

```json
{
  "data": {}
}
```

Collection:

```json
{
  "data": [],
  "links": {},
  "meta": {}
}
```

Mutation:

```json
{
  "message": "Operation completed",
  "data": {}
}
```

Delete без body допустим как `204 No Content`, но для admin UI предпочтителен JSON с `message`, чтобы Next client мог показывать toast.

### Pagination

Единый contract для списков:

Query params:

| Param | Type | Default | Rules |
|---|---:|---:|---|
| `page` | integer | `1` | min `1` |
| `per_page` | integer | domain-specific | min `1`, max `100` |

Response meta:

```json
{
  "meta": {
    "current_page": 1,
    "per_page": 20,
    "from": 1,
    "to": 20,
    "total": 120,
    "last_page": 6
  },
  "links": {
    "first": "https://...",
    "last": "https://...",
    "prev": null,
    "next": "https://..."
  }
}
```

### Sorting

Если endpoint поддерживает sort:

- `sort` — field name;
- `direction` — `asc` или `desc`.

Unsupported sort должен возвращать `422`, а не молча игнорироваться.

### Common resource shapes

#### Admin user compact

```json
{
  "id": 1,
  "name": "Admin",
  "email": "admin@example.com",
  "avatar": null,
  "roles": ["admin"],
  "is_banned": false,
  "ban_reason": null,
  "last_login_at": "2026-05-25T10:00:00Z",
  "last_login_ip": "127.0.0.1",
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": "2026-05-01T10:00:00Z"
}
```

#### Anime admin resource

```json
{
  "id": 1,
  "title": "Anime title",
  "slug": "anime-title",
  "description": "...",
  "poster_url": "https://...",
  "rating": 8.5,
  "status": "ongoing",
  "type": "tv",
  "year": 2024,
  "nsfw_flag": false,
  "external_id": "123",
  "external_source": "anilist",
  "episodes_count": 12,
  "tags": [
    { "id": 1, "name": "Action", "slug": "action" }
  ],
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": "2026-05-01T10:00:00Z"
}
```

#### Episode admin resource

```json
{
  "id": 1,
  "anime_id": 10,
  "episode_number": 1,
  "season_number": 1,
  "title": "Episode title",
  "duration": 1440,
  "player_url": "https://...",
  "player_iframe": "<iframe ...></iframe>",
  "anime": {
    "id": 10,
    "title": "Anime title",
    "slug": "anime-title"
  },
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": "2026-05-01T10:00:00Z"
}
```

#### Tag admin resource

```json
{
  "id": 1,
  "name": "Action",
  "slug": "action",
  "anime_count": 42,
  "created_at": "2026-05-01T10:00:00Z",
  "updated_at": "2026-05-01T10:00:00Z"
}
```

---

## 1. Admin authentication

### 1.1 Login

```http
POST /api/v1/admin/auth/login
```

Auth requirements: guest allowed.

Request:

```json
{
  "email": "admin@example.com",
  "password": "secret",
  "remember": true
}
```

Validation:

| Field | Rules |
|---|---|
| `email` | required, email |
| `password` | required, string |
| `remember` | optional, boolean |

Behavior:

1. Validate credentials.
2. Authenticate user.
3. Check `hasRole('admin')`.
4. If user is not admin, revoke/avoid session/token and return `403`.
5. Update `last_login_at`, `last_login_ip`.
6. Write audit log action `admin_login`.

Response `200` token mode:

```json
{
  "message": "Logged in",
  "data": {
    "token": "plain-text-token",
    "token_type": "Bearer",
    "user": {
      "id": 1,
      "name": "Admin",
      "email": "admin@example.com",
      "roles": ["admin"]
    }
  }
}
```

Response `200` cookie mode:

```json
{
  "message": "Logged in",
  "data": {
    "user": {
      "id": 1,
      "name": "Admin",
      "email": "admin@example.com",
      "roles": ["admin"]
    }
  }
}
```

Errors:

- `401` invalid credentials;
- `403` authenticated but not admin;
- `422` invalid payload.

### 1.2 Logout

```http
POST /api/v1/admin/auth/logout
```

Auth requirements: authenticated admin.

Request: empty JSON object or no body.

Behavior:

- revoke current token or invalidate current admin session;
- write audit log action `admin_logout`.

Response:

```json
{
  "message": "Logged out"
}
```

### 1.3 Me

```http
GET /api/v1/admin/auth/me
```

Auth requirements: authenticated admin.

Response:

```json
{
  "data": {
    "id": 1,
    "name": "Admin",
    "email": "admin@example.com",
    "avatar": null,
    "roles": ["admin"],
    "permissions": ["admin.access"],
    "is_admin": true
  }
}
```

### 1.4 Role checking contract

Required backend guard logic:

```text
authenticated && user.hasRole('admin')
```

Next admin client must not rely only on local role flags. It should:

1. call `/auth/me` on app boot/protected layout load;
2. redirect to `/login` on `401`;
3. show forbidden state or redirect on `403`;
4. clear local auth state on logout or `401` from any protected request.

---

## 2. Dashboard

### 2.1 Dashboard summary

```http
GET /api/v1/admin/dashboard
```

Auth requirements: authenticated admin.

Pagination/filter/sort: none.

Response:

```json
{
  "data": {
    "summary": {
      "total_anime": 100,
      "total_episodes": 1200,
      "total_tags": 50,
      "total_users": 300
    },
    "recent_imports": [
      {
        "id": 1,
        "import_type": "update",
        "status": "completed",
        "started_at": "2026-05-25T10:00:00Z",
        "completed_at": "2026-05-25T10:10:00Z",
        "created_at": "2026-05-25T10:00:00Z"
      }
    ],
    "anime_by_status": [
      { "status": "ongoing", "count": 20 }
    ],
    "anime_by_type": [
      { "type": "tv", "count": 80 }
    ],
    "latest_anime": []
  }
}
```

Implementation note: keep chart-ready arrays rather than keyed objects to simplify Next chart components.

---

## 3. User management

### 3.1 List users

```http
GET /api/v1/admin/users
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 20 | page size |
| `search` | string | max 255 | search by `name` or `email` |
| `role` | string | optional | future filter by role |
| `is_banned` | boolean | optional | ban filter |
| `sort` | string | `created_at`, `name`, `email`, `last_login_at` | default `created_at` |
| `direction` | string | `asc`, `desc` | default `desc` |

Response:

```json
{
  "data": [
    {
      "id": 1,
      "name": "User",
      "email": "user@example.com",
      "roles": ["user"],
      "is_banned": false,
      "ban_reason": null,
      "created_at": "2026-05-01T10:00:00Z"
    }
  ],
  "links": {},
  "meta": {}
}
```

### 3.2 Show user

```http
GET /api/v1/admin/users/{id}
```

Auth requirements: authenticated admin.

Path params:

| Param | Rules |
|---|---|
| `id` | existing user id |

Response:

```json
{
  "data": {
    "id": 1,
    "name": "User",
    "email": "user@example.com",
    "avatar": null,
    "bio": null,
    "custom_status": null,
    "roles": ["user"],
    "is_online": false,
    "is_premium": false,
    "is_banned": false,
    "ban_reason": null,
    "last_login_at": null,
    "last_login_ip": null,
    "created_at": "2026-05-01T10:00:00Z",
    "updated_at": "2026-05-01T10:00:00Z",
    "stats": {
      "comments_count": 0,
      "favorites_count": 0,
      "ratings_count": 0,
      "watch_history_count": 0
    }
  }
}
```

### 3.3 Ban user

```http
POST /api/v1/admin/users/{id}/ban
```

Auth requirements: authenticated admin.

Request:

```json
{ "reason": "Spam" }
```

Validation:

| Field | Rules |
|---|---|
| `reason` | required, string, max 255 |

Behavior:

- set `is_banned=true`;
- set `ban_reason`;
- write audit log `ban_user`.

Response:

```json
{
  "message": "User banned",
  "data": {}
}
```

### 3.4 Unban user

```http
POST /api/v1/admin/users/{id}/unban
```

Auth requirements: authenticated admin.

Request: empty.

Behavior:

- set `is_banned=false`;
- clear `ban_reason`;
- write audit log `unban_user`.

Response:

```json
{
  "message": "User unbanned",
  "data": {}
}
```

### 3.5 Delete user

```http
DELETE /api/v1/admin/users/{id}
```

Auth requirements: authenticated admin.

Validation/guards:

- `id` must exist;
- recommended: forbid deleting the currently authenticated admin account;
- recommended: require an extra confirmation flag if deleting an admin user.

Request optional:

```json
{ "confirm": true }
```

Response:

```json
{ "message": "User deleted" }
```

---

## 4. Anime management

### 4.1 List anime

```http
GET /api/v1/admin/anime
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 20 | page size |
| `search` | string | max 255 | search by title/slug |
| `status` | string | `planned`, `ongoing`, `finished`, `paused` | status filter |
| `type` | string | `tv`, `movie`, `ova`, `ona`, `special`, `music` | type filter |
| `tag_id` | integer | exists tags | optional tag filter |
| `nsfw_flag` | boolean | optional | NSFW filter |
| `sort` | string | `created_at`, `updated_at`, `title`, `rating`, `year`, `episodes_count` | default `created_at` |
| `direction` | string | `asc`, `desc` | default `desc` |

Response:

```json
{
  "data": [],
  "links": {},
  "meta": {}
}
```

### 4.2 Create anime

```http
POST /api/v1/admin/anime
```

Auth requirements: authenticated admin.

Request:

```json
{
  "title": "Anime title",
  "description": "Description",
  "poster_url": "https://example.com/poster.jpg",
  "rating": 8.5,
  "status": "ongoing",
  "type": "tv",
  "nsfw_flag": false,
  "tags": [1, 2]
}
```

Validation:

| Field | Rules |
|---|---|
| `title` | required, string, max 255 |
| `description` | nullable, string |
| `poster_url` | nullable, url |
| `rating` | nullable, numeric, between 0 and 10 |
| `status` | required, in `planned`, `ongoing`, `finished`, `paused` |
| `type` | required, in `tv`, `movie`, `ova`, `ona`, `special`, `music` |
| `nsfw_flag` | nullable, boolean |
| `tags` | nullable, array |
| `tags.*` | integer, exists `tags.id` |

Behavior:

- generate `slug` from title using current backend rules;
- sync tags if provided;
- write audit log `create_anime`.

Response `201`:

```json
{
  "message": "Anime created",
  "data": {}
}
```

### 4.3 Show anime

```http
GET /api/v1/admin/anime/{id}
```

Auth requirements: authenticated admin.

Response:

```json
{
  "data": {
    "id": 1,
    "title": "Anime title",
    "tags": [],
    "episodes": [
      {
        "id": 10,
        "episode_number": 1,
        "title": "Episode 1",
        "duration": 1440,
        "player_url": "https://..."
      }
    ]
  }
}
```

### 4.4 Update anime

```http
PUT /api/v1/admin/anime/{id}
```

Auth requirements: authenticated admin.

Request and validation: same as create.

Behavior:

- regenerate slug if title changed;
- if `tags` is provided, sync tags;
- if `tags` is an empty array, detach all tags;
- write audit log `update_anime`.

Response:

```json
{
  "message": "Anime updated",
  "data": {}
}
```

### 4.5 Delete anime

```http
DELETE /api/v1/admin/anime/{id}
```

Auth requirements: authenticated admin.

Behavior:

- if anime has `external_id`, add it to blacklist with `external_source` fallback `anilist`;
- delete anime;
- write audit log `delete_anime`.

Response:

```json
{ "message": "Anime deleted and blacklisted" }
```

### 4.6 Bulk delete anime

```http
POST /api/v1/admin/anime/bulk-delete
```

Auth requirements: authenticated admin.

Request:

```json
{ "ids": [1, 2, 3] }
```

Validation:

| Field | Rules |
|---|---|
| `ids` | required, array, min 1 |
| `ids.*` | integer, distinct, exists `anime.id` |

Behavior:

- delete selected anime;
- blacklist each anime with external id;
- write audit log `bulk_delete_anime`.

Response:

```json
{
  "message": "Anime records deleted",
  "data": {
    "deleted_count": 3,
    "blacklisted_count": 2
  }
}
```

---

## 5. Tag management

### 5.1 List tags

```http
GET /api/v1/admin/tags
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 50 | page size |
| `search` | string | max 100 | search by name |
| `sort` | string | `name`, `anime_count`, `created_at` | default `name` |
| `direction` | string | `asc`, `desc` | default `asc` |

Response:

```json
{
  "data": [],
  "links": {},
  "meta": {}
}
```

### 5.2 Create tag

```http
POST /api/v1/admin/tags
```

Auth requirements: authenticated admin.

Request:

```json
{ "name": "Action" }
```

Validation:

| Field | Rules |
|---|---|
| `name` | required, string, max 100, unique `tags.name` |

Behavior:

- generate `slug` from name;
- write audit log `create_tag`.

Response `201`:

```json
{
  "message": "Tag created",
  "data": {}
}
```

### 5.3 Show tag

```http
GET /api/v1/admin/tags/{id}
```

Auth requirements: authenticated admin.

Response:

```json
{
  "data": {
    "id": 1,
    "name": "Action",
    "slug": "action",
    "anime_count": 42
  }
}
```

### 5.4 Update tag

```http
PUT /api/v1/admin/tags/{id}
```

Auth requirements: authenticated admin.

Request:

```json
{ "name": "Action" }
```

Validation:

| Field | Rules |
|---|---|
| `name` | required, string, max 100, unique `tags.name` ignoring current id |

Behavior:

- regenerate slug;
- write audit log `update_tag`.

Response:

```json
{
  "message": "Tag updated",
  "data": {}
}
```

### 5.5 Delete tag

```http
DELETE /api/v1/admin/tags/{id}
```

Auth requirements: authenticated admin.

Behavior:

- delete tag;
- write audit log `delete_tag`.

Response:

```json
{ "message": "Tag deleted" }
```

---

## 6. Episode management and imports

### 6.1 List episodes

```http
GET /api/v1/admin/episodes
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 50 | page size |
| `anime_id` | integer | exists `anime.id` | filter by anime |
| `search` | string | max 255 | search by anime title |
| `sort` | string | `anime_id`, `episode_number`, `created_at`, `updated_at` | default composite behavior |
| `direction` | string | `asc`, `desc` | default `desc` |

Default ordering must preserve current Blade behavior where possible:

```text
anime_id desc, episode_number desc
```

Response:

```json
{
  "data": [],
  "links": {},
  "meta": {},
  "included": {
    "filters": {
      "selected_anime": null
    }
  }
}
```

Do not include full `allAnimes` list by default for large catalogs. Use `GET /api/v1/admin/anime?per_page=...&search=...` for selector options.

### 6.2 Show episode

```http
GET /api/v1/admin/episodes/{id}
```

Auth requirements: authenticated admin.

Response:

```json
{
  "data": {}
}
```

### 6.3 Update episode

```http
PUT /api/v1/admin/episodes/{id}
```

Auth requirements: authenticated admin.

Request:

```json
{
  "episode_number": 1,
  "title": "Episode title",
  "duration": 1440,
  "player_url": "https://example.com/player",
  "player_iframe": "<iframe ...></iframe>"
}
```

Validation:

| Field | Rules |
|---|---|
| `episode_number` | required, integer, min 1 |
| `title` | nullable, string, max 255 |
| `duration` | nullable, integer, min 0 |
| `player_url` | nullable, url, max 2048 |
| `player_iframe` | nullable, string |

Behavior:

- update episode;
- write audit log `update_episode`.

Response:

```json
{
  "message": "Episode updated",
  "data": {}
}
```

### 6.4 Delete episode

```http
DELETE /api/v1/admin/episodes/{id}
```

Auth requirements: authenticated admin.

Response:

```json
{ "message": "Episode deleted" }
```

### 6.5 Import episodes for all anime

```http
POST /api/v1/admin/episodes/import-all
```

Auth requirements: authenticated admin.

Request optional:

```json
{ "force": false }
```

Behavior:

- dispatch episode import job for every anime;
- do not block request until imports finish.

Response `202`:

```json
{
  "message": "Mass episode import started",
  "data": {
    "queued_count": 100
  }
}
```

### 6.6 Import episodes for one anime

```http
POST /api/v1/admin/episodes/{animeId}/import
```

Auth requirements: authenticated admin.

Path params:

| Param | Rules |
|---|---|
| `animeId` | exists `anime.id` |

Response `202`:

```json
{
  "message": "Episode import started",
  "data": {
    "anime_id": 1
  }
}
```

### 6.7 Bulk import episodes

```http
POST /api/v1/admin/episodes/bulk-import
```

Auth requirements: authenticated admin.

Request:

```json
{ "anime_ids": [1, 2, 3] }
```

Validation:

| Field | Rules |
|---|---|
| `anime_ids` | nullable, array |
| `anime_ids.*` | integer, exists `anime.id` |

Behavior:

- if `anime_ids` is empty/missing, preserve current behavior and start import for all anime;
- otherwise dispatch jobs for selected anime.

Response `202`:

```json
{
  "message": "Bulk episode import started",
  "data": {
    "queued_count": 3,
    "all_anime": false
  }
}
```

---

## 7. Comment moderation

### 7.1 List comments

```http
GET /api/v1/admin/comments
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 20 | page size |
| `status` | string | `approved`, `rejected`, `pending` | moderation status |
| `search` | string | max 255 | search in comment body |
| `anime_id` | integer | exists `anime.id` | optional future filter |
| `user_id` | integer | exists `users.id` | optional future filter |
| `sort` | string | `created_at`, `updated_at` | default `created_at` |
| `direction` | string | `asc`, `desc` | default `desc` |

Current Blade code stores approval as boolean `is_approved`. If no tri-state pending exists in DB, `pending` must be documented as future-only and either not accepted until DB support exists, or mapped explicitly in implementation notes.

Response:

```json
{
  "data": [
    {
      "id": 1,
      "comment": "Text",
      "is_approved": true,
      "user": { "id": 1, "name": "User", "email": "user@example.com" },
      "anime": { "id": 1, "title": "Anime title", "slug": "anime-title" },
      "created_at": "2026-05-01T10:00:00Z"
    }
  ],
  "links": {},
  "meta": {}
}
```

### 7.2 Approve comment

```http
POST /api/v1/admin/comments/{id}/approve
```

Auth requirements: authenticated admin.

Behavior: set `is_approved=true`.

Response:

```json
{
  "message": "Comment approved",
  "data": {}
}
```

### 7.3 Reject comment

```http
POST /api/v1/admin/comments/{id}/reject
```

Auth requirements: authenticated admin.

Behavior: set `is_approved=false`.

Response:

```json
{
  "message": "Comment rejected",
  "data": {}
}
```

### 7.4 Delete comment

```http
DELETE /api/v1/admin/comments/{id}
```

Auth requirements: authenticated admin.

Behavior:

- delete comment;
- preserve current counter behavior, but implementation must guard against missing anime before decrementing `comments_count`.

Response:

```json
{ "message": "Comment deleted" }
```

---

## 8. Import management

### 8.1 Imports dashboard

```http
GET /api/v1/admin/imports/dashboard
```

Auth requirements: authenticated admin.

Response:

```json
{
  "data": {
    "latest_import": {
      "id": 1,
      "import_type": "update",
      "status": "completed",
      "started_at": "2026-05-25T10:00:00Z",
      "completed_at": "2026-05-25T10:10:00Z"
    },
    "stats": {
      "total_imports": 10,
      "successful_imports": 8,
      "failed_imports": 1,
      "running_imports": 1
    }
  }
}
```

### 8.2 Run import

```http
POST /api/v1/admin/imports/run
```

Auth requirements: authenticated admin.

Request:

```json
{ "type": "update" }
```

Validation:

| Field | Rules |
|---|---|
| `type` | required, in `initial`, `update` |

Behavior:

- create `ImportLog` with `status=running`;
- dispatch import job;
- write audit log `run_import`.

Response `202`:

```json
{
  "message": "Import started",
  "data": {
    "import_log_id": 1,
    "type": "update",
    "status": "running"
  }
}
```

### 8.3 Import logs

```http
GET /api/v1/admin/imports/logs
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 20 | page size |
| `status` | string | `running`, `completed`, `failed` | optional |
| `type` | string | `initial`, `update` | optional |
| `sort` | string | `created_at`, `started_at`, `completed_at` | default `created_at` |
| `direction` | string | `asc`, `desc` | default `desc` |

Response:

```json
{
  "data": [
    {
      "id": 1,
      "import_type": "update",
      "status": "completed",
      "message": null,
      "started_at": "2026-05-25T10:00:00Z",
      "completed_at": "2026-05-25T10:10:00Z",
      "created_at": "2026-05-25T10:00:00Z"
    }
  ],
  "links": {},
  "meta": {}
}
```

---

## 9. Audit logs

### 9.1 List audit logs

```http
GET /api/v1/admin/audit-logs
```

Auth requirements: authenticated admin.

Query params:

| Param | Type | Rules | Description |
|---|---|---|---|
| `page` | integer | min 1 | page |
| `per_page` | integer | 1-100, default 50 | page size |
| `action` | string | max 100 | filter by action |
| `user_id` | integer | exists `users.id` | filter by actor |
| `date_from` | date | ISO date | inclusive |
| `date_to` | date | ISO date, after/equal `date_from` | inclusive |
| `sort` | string | `created_at`, `action`, `user_id` | default `created_at` |
| `direction` | string | `asc`, `desc` | default `desc` |

Response:

```json
{
  "data": [
    {
      "id": 1,
      "user_id": 1,
      "action": "admin_login",
      "description": "Admin logged in via web panel",
      "ip_address": "127.0.0.1",
      "user_agent": "Mozilla/5.0",
      "user": {
        "id": 1,
        "name": "Admin",
        "email": "admin@example.com"
      },
      "created_at": "2026-05-25T10:00:00Z"
    }
  ],
  "links": {},
  "meta": {},
  "included": {
    "actions": ["admin_login", "admin_logout", "create_anime"]
  }
}
```

---

## Implementation notes for next coding phase

- Prefer Laravel API Resources for stable response shapes.
- Keep legacy Blade routes/controllers/views until migration parity is explicitly verified.
- Do not rename existing public API endpoints.
- Avoid exposing password hashes, remember tokens, raw internal exception traces, or secrets in admin responses.
- All admin mutations should write audit logs where current Blade behavior already does; missing audit logs can be added later as a separate hardening wave.
- Jobs/import endpoints should return `202 Accepted` when work is queued asynchronously.
