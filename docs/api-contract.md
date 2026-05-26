# API contract

## Base URL

Frontend обращается к backend через Next.js proxy:

```text
/api/external/*
```

Backend API prefix:

```text
/api/v1
```

Пример:

```text
Frontend: /api/external/anime
Backend:  /api/v1/public/anime
```

## Auth

Private endpoints требуют Bearer token:

```http
Authorization: Bearer <token>
Accept: application/json
Content-Type: application/json
```

Токен выдается endpoints:

- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`

Frontend хранит token в `localStorage.userToken`.

## Public endpoints

### Anime list

```http
GET /api/v1/public/anime
```

Query params:

| Param | Type | Description |
|---|---|---|
| `page` | integer | Номер страницы |
| `per_page` | integer | Размер страницы, 1–100 |
| `type` | string | `tv`, `movie`, `ova`, `ona`, `special`, `music` |
| `status` | string | `planned`, `ongoing`, `finished`, `paused`; aliases: `releasing`, `upcoming`, `anons` |
| `year` | integer | Год |
| `search` | string | Поиск по title/slug |
| `genre` | string | Slug тега |
| `sort` | string | `smart`, `rating`, `popularity`, `newest`, `title` |

Response:

```json
{
  "data": [
    {
      "id": 1,
      "title": "Anime title",
      "slug": "anime-title",
      "poster_url": "https://...",
      "rating": 8.5,
      "year": 2024,
      "type": "tv",
      "status": "ongoing",
      "tags": []
    }
  ],
  "links": {},
  "meta": {}
}
```

### Anime details

```http
GET /api/v1/public/anime/{anime}
```

Response:

```json
{
  "data": {
    "id": 1,
    "title": "Anime title",
    "description": "...",
    "poster_url": "https://...",
    "episodes_count": 12,
    "tags": []
  }
}
```

### Anime banner

```http
GET /api/v1/public/anime/{anime}/banner
```

Response:

```json
{
  "banner": "https://...",
  "cover": "https://..."
}
```

### Anime episodes

```http
GET /api/v1/public/anime/{anime}/episodes
```

Response may be grouped by source:

```json
{
  "data": {
    "AniLibria": [
      {
        "id": 10,
        "episode_number": 1,
        "title": "Episode 1"
      }
    ]
  }
}
```

### Player sources

```http
GET /api/v1/public/anime/{anime}/episodes/{episodeNumber}/sources
```

Response:

```json
{
  "data": [
    {
      "source": "AniLibria",
      "quality": "hls",
      "url": "https://..."
    }
  ]
}
```

### Tags

```http
GET /api/v1/public/tags
```

Query params:

| Param | Type | Description |
|---|---|---|
| `search` | string | Search by name/slug |

Response:

```json
{
  "success": true,
  "data": [
    { "id": 1, "name": "Action", "slug": "action" }
  ],
  "total": 1
}
```

### Schedule

```http
GET /api/v1/public/schedule
```

Response:

```json
{
  "success": true,
  "data": {
    "0": [],
    "1": [],
    "2": [],
    "3": [],
    "4": [],
    "5": [],
    "6": []
  }
}
```

Days are indexed from Monday `0` to Sunday `6`.

### Public user statistics

```http
GET /api/v1/public/users/{userId}/statistics
```

## Auth endpoints

### Register

```http
POST /api/v1/auth/register
```

Body:

```json
{
  "name": "User",
  "email": "user@example.com",
  "password": "password",
  "password_confirmation": "password"
}
```

### Login

```http
POST /api/v1/auth/login
```

Body:

```json
{
  "email": "user@example.com",
  "password": "password"
}
```

Response:

```json
{
  "user": {},
  "token": "plain-text-sanctum-token"
}
```

### Current user

```http
GET /api/v1/user
```

Auth: required.

### Logout

```http
POST /api/v1/auth/logout
```

Auth: required.

## Private profile/statistics endpoints

### Full profile

```http
GET /api/v1/profile/me
```

### Update profile

```http
PUT /api/v1/profile/me
```

Body:

```json
{
  "name": "New name",
  "bio": "About me",
  "custom_status": "Watching anime"
}
```

### Upload avatar

```http
POST /api/v1/profile/me/avatar
```

Content-Type: `multipart/form-data`.

### My statistics

```http
GET /api/v1/statistics/me
GET /api/v1/statistics/me/episodes-summary
```

## Anime list endpoints

### Update anime status

```http
POST /api/v1/anime/{anime}/status
```

Body:

```json
{
  "status": "watching"
}
```

Known statuses:

- `watching`
- `planned`
- `completed`
- `on_hold`
- `dropped`
- `not_watching` — detach from list

### User anime status

```http
GET /api/v1/anime/{anime}/user-status
```

### Update watched episodes

```http
PATCH /api/v1/anime/{anime}/episodes-watched/{episodesWatched}
```

### My anime list

```http
GET /api/v1/my-anime-list/{status?}
```

## Favorites

```http
GET    /api/v1/favorites
POST   /api/v1/favorites
DELETE /api/v1/favorites/{animeId}
GET    /api/v1/favorites/{animeId}/check
```

Create body:

```json
{
  "anime_id": 1
}
```

## Watch history

```http
GET    /api/v1/watch-history
POST   /api/v1/watch-history
GET    /api/v1/watch-history/{id}
DELETE /api/v1/watch-history/{id}
GET    /api/v1/watch-history/anime/{animeId}/history
GET    /api/v1/watch-history/anime/{animeId}/last-episode
```

Store body:

```json
{
  "episode_id": 10,
  "progress": 120,
  "delta_time": 30,
  "completed": false
}
```

Validation notes:

- `delta_time` required;
- `delta_time` min `0`, max `300`;
- endpoint requires auth.

## Ratings

```http
GET    /api/v1/ratings
POST   /api/v1/ratings
DELETE /api/v1/ratings/{rating}
GET    /api/v1/ratings/anime/{animeId}
```

Create body:

```json
{
  "anime_id": 1,
  "score": 9
}
```

## Comments

```http
GET    /api/v1/public/anime/{anime}/comments
GET    /api/v1/my-comments
POST   /api/v1/comments
PUT    /api/v1/comments/{comment}
DELETE /api/v1/comments/{comment}
```

Create body:

```json
{
  "anime_id": 1,
  "content": "Great anime"
}
```

## Friends

```http
GET  /api/v1/friends
GET  /api/v1/friends/requests
GET  /api/v1/friends/requests/count
POST /api/v1/friends/{userId}
POST /api/v1/friends/{userId}/accept
POST /api/v1/friends/{userId}/decline
GET  /api/v1/friends/{userId}/status
GET  /api/v1/users/search?q={query}
```

## Watch Party

```http
POST   /api/v1/watch-party
GET    /api/v1/watch-party/{code}
POST   /api/v1/watch-party/{code}/join
POST   /api/v1/watch-party/{code}/leave
POST   /api/v1/watch-party/{code}/sync
POST   /api/v1/watch-party/{code}/message
GET    /api/v1/watch-party/{code}/messages
POST   /api/v1/watch-party/{code}/invite
DELETE /api/v1/watch-party/{code}
```

Create body:

```json
{
  "anime_id": 1,
  "episode_number": 1
}
```

Sync body:

```json
{
  "current_time": 120.5,
  "is_playing": true,
  "episode_number": 1
}
```

Message body:

```json
{
  "message": "hello"
}
```

Invite body:

```json
{
  "friend_id": 2
}
```

## Broadcast auth

Frontend uses:

```text
/api/external/broadcasting/auth
```

Backend target:

```text
/api/v1/broadcasting/auth
```

Auth: required.

This endpoint authorizes Reverb private/presence channels.

## Payment

```http
POST /api/v1/payment/premium
```

Auth: required.

## Error format

For API requests, backend returns JSON errors through `bootstrap/app.php` exception handling.

Validation error:

```json
{
  "message": "Validation failed",
  "errors": {}
}
```

Unauthenticated:

```json
{
  "message": "Unauthenticated"
}
```

Not found:

```json
{
  "message": "Resource not found"
}
```

Unauthorized:

```json
{
  "message": "Unauthorized"
}
```
