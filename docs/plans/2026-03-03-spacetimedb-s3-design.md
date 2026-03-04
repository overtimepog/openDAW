# SpacetimeDB + S3 Collaboration Design

**Date:** 2026-03-03
**Status:** Approved
**Approach:** SpacetimeDB-First with WebRTC Sidecar (Approach A)

## Goals

- Excalidraw-like collaboration: shareable links, no auth, zero friction
- SpacetimeDB as primary sync backend (YSync deprecated over time)
- S3 optional for cloud asset persistence, OPFS as local fallback
- WebRTC P2P for binary asset transfer between collaborators
- Configurable SpacetimeDB endpoint (default central instance, self-hostable)
- Cursors + names for presence
- TDD enforcement throughout implementation

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        openDAW Client                       │
│                                                             │
│  ┌───────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
│  │ BoxGraph  │  │ Presence │  │  Asset    │  │   S3      │  │
│  │   Sync    │  │  Service │  │ Transfer │  │  Service  │  │
│  └─────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬─────┘  │
│        │              │             │              │         │
│  ┌─────┴──────────────┴─────┐  ┌───┴──────────────┴──────┐  │
│  │    StdbSyncBackend       │  │    AssetTransport        │  │
│  │  (SpacetimeDB client)    │  │  (WebRTC / S3 / OPFS)   │  │
│  └──────────┬───────────────┘  └───┬──────────┬──────────┘  │
└─────────────┼──────────────────────┼──────────┼─────────────┘
              │ WebSocket            │ WebRTC   │ HTTPS
              ▼                      ▼          ▼
     ┌────────────────┐        ┌─────────┐  ┌──────┐
     │  SpacetimeDB   │        │  Peers  │  │  S3  │
     │                │        │ (other  │  │(user │
     │ • BoxGraph     │        │ clients)│  │  own)│
     │ • Rooms        │        └─────────┘  └──────┘
     │ • Presence     │
     │ • Signaling    │
     └────────────────┘
```

Three communication channels:
1. **SpacetimeDB (WebSocket)** — state sync: BoxGraph changes, room management, presence, WebRTC signaling
2. **WebRTC (P2P)** — binary asset transfer between peers (audio samples, soundfonts)
3. **S3 (HTTPS)** — optional persistent cloud storage for assets (user-provided)

SpacetimeDB never stores binary audio data. It stores metadata (asset IDs, names, sizes) and acts as the signaling server for WebRTC.

## Rooms & Shareable Links

### SpacetimeDB Tables

| Table | Purpose |
|-------|---------|
| `room` | ID, created_at, is_persistent, creator_identity, last_active_at |
| `room_asset` | room_id, asset_id, name, size_bytes, mime_type, s3_url (optional), uploaded_by |
| `room_participant` | room_id, identity, display_name, color, joined_at |

### Ephemeral vs Persistent

- **Ephemeral** (default): Room created on "Share" click. Auto-deleted after all participants leave + a grace period (~30 min). No sign-up needed.
- **Persistent**: User clicks "Save session" to promote an ephemeral room. Room survives disconnections. Stored in SpacetimeDB indefinitely.

### URL Structure

```
https://opendaw.studio/r/{room_id}
```

- `room_id` is a short, URL-safe ID (nanoid-style, ~8 chars)
- No encryption key in URL hash — SpacetimeDB only stores metadata and BoxGraph state, no binary data to protect

### Join Flow

1. Joiner opens link → client connects to SpacetimeDB
2. SpacetimeDB returns room state (BoxGraph snapshot + asset registry + participant list)
3. Client hydrates BoxGraph, then resolves missing assets via the asset transport layer

## Asset Transport Layer

Assets resolve through a priority-ordered fallback chain:

```
Asset Request for UUID "abc123"
    │
    ├─ 1. OPFS (local)       → Already cached? Use it.
    │
    ├─ 2. S3 (if configured) → room_asset has s3_url? Fetch it.
    │
    └─ 3. WebRTC (P2P)       → Ask peers who have it. Transfer directly.
         │
         └─ Nobody has it    → Show "Missing asset" indicator
```

### WebRTC Signaling via SpacetimeDB

SpacetimeDB tables act as the signaling channel (SDP offers/answers/ICE candidates). No separate signaling server needed. Once WebRTC data channel is established, binary transfers happen directly between peers.

### Asset Registry Flow

1. Creator adds a sample → stored in OPFS → metadata written to `room_asset` table
2. If S3 is configured, upload to S3 → update `room_asset.s3_url`
3. Collaborator joins → reads `room_asset` list → tries OPFS → S3 → WebRTC in order
4. Downloaded assets are cached in OPFS for future sessions

### S3 Configuration (User-Provided)

- Settings panel: bucket name, region, access key, secret key, optional endpoint (for S3-compatible services like MinIO, R2)
- Stored locally only (never sent to SpacetimeDB)
- Client uploads/downloads directly to S3
- Only the resulting S3 URL is shared via `room_asset` table

## Presence System

### Table

| Field | Type | Purpose |
|-------|------|---------|
| identity | Identity | SpacetimeDB client identity |
| room_id | String | Which room |
| display_name | String | User-entered name |
| color | String | Auto-assigned from palette |
| cursor_x | f64 | Cursor X position |
| cursor_y | f64 | Cursor Y position |
| cursor_target | String | UI element ID (track, region, piano roll) |
| last_seen | Timestamp | For stale cleanup |

### Behavior

- Presence updates throttled to ~10Hz
- Color auto-assigned from palette on join (consistent across session)
- Display name entered on first join (stored in localStorage for future sessions)
- Stale presence cleaned up via SpacetimeDB scheduled reducer

### UI

- Colored cursor indicators overlaid on arrangement/track view
- Small name label next to each cursor
- Participant list in a collapsible panel (like Excalidraw's people panel)

## Client Architecture

### New Modules

| Location | Purpose |
|----------|---------|
| `packages/studio/core/src/stdb/` (existing) | Extend with room management, asset registry, WebRTC signaling |
| `packages/studio/core/src/assets/AssetTransport.ts` | OPFS/S3/WebRTC resolution layer |
| `packages/studio/core/src/assets/WebRTCTransfer.ts` | P2P binary transfer via WebRTC data channels |
| `packages/studio/core/src/assets/S3Service.ts` | S3 upload/download with user credentials |
| `packages/app/studio/src/ui/collab/` | Share dialog, participant panel, join screen, name prompt |

### Key Interfaces

```typescript
interface AssetTransport {
  resolve(assetId: UUID): Promise<ArrayBuffer | undefined>
  publish(assetId: UUID, data: ArrayBuffer, meta: AssetMeta): Promise<void>
}

interface S3Config {
  bucket: string
  region: string
  accessKeyId: string
  secretAccessKey: string
  endpoint?: string  // for S3-compatible services (MinIO, R2, etc.)
}
```

### Integration with Existing Code

- **StdbSyncBackend** (existing) — extend with room lifecycle
- **SampleStorage** (OPFS, existing) — AssetTransport wraps as first cache layer
- **StudioMenu.ts** — replace YSync menu items with SpacetimeDB collaboration options
- **StdbMapper** — already handles BoxGraph ↔ SpacetimeDB sync, no core sync changes needed

### What Stays Unchanged

- BoxGraph, Box, Field — core data model untouched
- Audio engine / worklet — no changes
- SampleService — still handles decoding/peaks, gets new source via AssetTransport
- Project bundle format — still works for local save/load, independent of collaboration

## SpacetimeDB Module Changes

### New Tables (Rust)

```rust
// room
// id: String (nanoid, 8 chars)
// creator_identity: Identity
// is_persistent: bool
// created_at: Timestamp
// last_active_at: Timestamp

// room_asset
// room_id: String
// asset_id: String (UUID)
// name: String
// size_bytes: u64
// mime_type: String
// s3_url: Option<String>
// uploaded_by: Identity

// webrtc_signal
// room_id: String
// from_identity: Identity
// to_identity: Identity
// signal_type: String (offer/answer/ice)
// payload: String (JSON)
// created_at: Timestamp
```

### New Reducers

- `create_room()` → generates nanoid, returns room_id
- `join_room(room_id)` → adds participant, returns current state
- `leave_room(room_id)` → removes participant, triggers cleanup if empty
- `promote_room(room_id)` → sets is_persistent = true
- `register_asset(room_id, asset_id, meta)` → adds to room_asset
- `update_asset_s3_url(room_id, asset_id, url)` → sets S3 URL
- `send_signal(room_id, to, signal_type, payload)` → WebRTC signaling
- `cleanup_stale_rooms()` → scheduled reducer, cleans ephemeral rooms after grace period

### Existing Tables Kept

`box_state`, `box_updated`, `presence` (extended with cursor fields), `user`, `project`, `project_member`, `invitation`
