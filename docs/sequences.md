# Sequence diagrams

This document describes key runtime scenarios in Aniyume.

## 1. Login

```mermaid
sequenceDiagram
    actor User
    participant UI as Next.js UI
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database

    User->>UI: Enter email/password
    UI->>Proxy: POST /api/external/auth/login
    Proxy->>API: POST /api/v1/auth/login
    API->>DB: Find user, verify password
    DB-->>API: User record
    API-->>Proxy: user + Sanctum token
    Proxy-->>UI: user + token
    UI->>UI: Save token to localStorage.userToken
    UI-->>User: Authenticated state
```

## 2. Open anime page

```mermaid
sequenceDiagram
    actor User
    participant Page as Anime page
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database
    participant Ext as External APIs

    User->>Page: Open /anime/{id}
    Page->>Proxy: GET /api/external/public/anime/{id}
    Proxy->>API: GET /api/v1/public/anime/{id}
    API->>DB: Load anime + tags
    DB-->>API: Anime data
    API-->>Page: Anime details

    Page->>Proxy: GET /api/external/public/anime/{id}/episodes
    Proxy->>API: GET /api/v1/public/anime/{id}/episodes
    API->>DB: Load episodes
    DB-->>API: Episodes
    API-->>Page: Episodes

    Page->>Proxy: GET /api/external/public/anime/{id}/banner
    Proxy->>API: GET /api/v1/public/anime/{id}/banner
    API->>Ext: Fetch/cache banner if needed
    Ext-->>API: Banner/cover
    API-->>Page: Banner data
```

## 3. Save watch progress

```mermaid
sequenceDiagram
    actor User
    participant Player as Video player
    participant Tracker as useWatchTracker
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database

    User->>Player: Watch episode
    Player->>Tracker: Progress event
    Tracker->>Tracker: Buffer delta_time
    Tracker->>Proxy: POST /api/external/watch-history
    Note over Tracker,Proxy: Authorization: Bearer token
    Proxy->>API: POST /api/v1/watch-history
    API->>API: Validate episode_id, progress, delta_time
    API->>DB: Upsert watch history/progress
    API->>DB: Update aggregate anime_user if needed
    DB-->>API: Saved
    API-->>Tracker: Success
```

## 4. Add anime to list

```mermaid
sequenceDiagram
    actor User
    participant UI as Anime actions
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database

    User->>UI: Select status watching/planned/etc.
    UI->>Proxy: POST /api/external/anime/{anime}/status
    Proxy->>API: POST /api/v1/anime/{anime}/status
    API->>DB: sync/detach anime_user pivot
    DB-->>API: Updated pivot
    API-->>UI: Current status
```

## 5. Create Watch Party

```mermaid
sequenceDiagram
    actor Host
    participant UI as Watch Party UI
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database
    participant Reverb as Laravel Reverb

    Host->>UI: Create room
    UI->>Proxy: POST /api/external/watch-party
    Proxy->>API: POST /api/v1/watch-party
    API->>DB: Create room + host participant
    DB-->>API: Room code
    API-->>UI: Room data
    UI->>Reverb: Join presence channel watch-party.{code}
    Reverb-->>UI: Presence members
```

## 6. Watch Party player sync

```mermaid
sequenceDiagram
    actor Host
    participant HostUI as Host player
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant Event as PlayerSyncEvent
    participant Reverb as Reverb
    participant GuestUI as Guest player

    Host->>HostUI: Play/pause/seek
    HostUI->>Proxy: POST /api/external/watch-party/{code}/sync
    Proxy->>API: POST /api/v1/watch-party/{code}/sync
    API->>Event: Dispatch PlayerSyncEvent
    Event->>Reverb: BroadcastNow .player.sync
    Reverb-->>GuestUI: player.sync payload
    GuestUI->>GuestUI: Apply current_time/is_playing
```

## 7. Watch Party chat

```mermaid
sequenceDiagram
    actor User
    participant UI as Watch Party chat
    participant Proxy as /api/external proxy
    participant API as Laravel API
    participant DB as Database
    participant Event as ChatMessageEvent
    participant Reverb as Reverb
    participant Others as Other participants

    User->>UI: Send message
    UI->>UI: Optimistic message
    UI->>Proxy: POST /api/external/watch-party/{code}/message
    Proxy->>API: POST /api/v1/watch-party/{code}/message
    API->>DB: Store message
    DB-->>API: Message id
    API->>Event: Dispatch ChatMessageEvent
    Event->>Reverb: BroadcastNow .chat.message
    Reverb-->>Others: Chat payload
    API-->>UI: Stored message
    UI->>UI: Replace optimistic message
```

## 8. Friend invite to Watch Party

```mermaid
sequenceDiagram
    actor Host
    participant UI as Watch Party UI
    participant API as Laravel API
    participant Event as FriendInviteEvent
    participant Reverb as Reverb
    participant Friend as Friend browser

    Host->>UI: Invite friend
    UI->>API: POST /api/v1/watch-party/{code}/invite
    API->>API: Check friendship/access
    API->>Event: Dispatch FriendInviteEvent
    Event->>Reverb: BroadcastNow private user.{friendId}
    Reverb-->>Friend: friend.invite
    Friend-->>Friend: Show invite notification
```

## 9. Admin episode import

```mermaid
sequenceDiagram
    actor Admin
    participant AdminUI as Admin panel
    participant Web as Laravel web routes
    participant Job as EpisodesImportJob
    participant Service as EpisodeImportService
    participant Ext as External video APIs
    participant DB as Database

    Admin->>AdminUI: Click import episodes
    AdminUI->>Web: POST /admin/episodes/{anime}/import
    Web->>Job: Dispatch EpisodesImportJob
    Job->>Service: importForSingleAnime(anime)
    Service->>Ext: Fetch episodes/sources
    Ext-->>Service: Episode data
    Service->>DB: Create/update episodes
    DB-->>Service: Saved
```

## 10. Scheduled import

```mermaid
sequenceDiagram
    participant Cron as System cron
    participant Scheduler as Laravel scheduler
    participant Cmd as Artisan command
    participant Service as Import service
    participant Ext as External API
    participant DB as Database

    Cron->>Scheduler: php artisan schedule:run
    Scheduler->>Cmd: import:anime / import:episodes
    Cmd->>Service: Run import
    Service->>Ext: Fetch external data
    Ext-->>Service: Data
    Service->>DB: Upsert records + import_logs
```
