# SpacetimeDB + S3 Collaboration Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Excalidraw-like real-time collaboration to openDAW using SpacetimeDB for state sync, WebRTC for P2P asset transfer, and optional user-provided S3 for cloud storage.

**Architecture:** SpacetimeDB handles all state (BoxGraph sync, rooms, presence, WebRTC signaling). WebRTC data channels transfer binary assets P2P. S3 is an optional persistent asset store. Assets resolve via a fallback chain: OPFS -> S3 -> WebRTC -> missing.

**Tech Stack:** SpacetimeDB (Rust server module + TypeScript client SDK), WebRTC (native browser APIs), AWS S3 SDK (user-provided credentials), Vitest (testing)

**Design Doc:** `docs/plans/2026-03-03-spacetimedb-s3-design.md`

---

## Phase 1: SpacetimeDB Server Module

### Task 1: Scaffold SpacetimeDB Rust Module

**Files:**
- Create: `packages/server/spacetimedb-module/Cargo.toml`
- Create: `packages/server/spacetimedb-module/src/lib.rs`

**Step 1: Create Cargo.toml**

```toml
[package]
name = "opendaw-spacetimedb"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
spacetimedb = "1.0"
log = "0.4"
rand = "0.8"
```

**Step 2: Create minimal lib.rs with a smoke-test table**

```rust
use spacetimedb::{spacetimedb, Identity, Timestamp, ReducerContext, Table, reducer};

#[spacetimedb::table(name = room, public)]
pub struct Room {
    #[primary_key]
    pub id: String,
    pub creator_identity: Identity,
    pub is_persistent: bool,
    pub created_at: Timestamp,
    pub last_active_at: Timestamp,
}

#[reducer]
pub fn ping(ctx: &ReducerContext) -> Result<(), String> {
    log::info!("ping from {:?}", ctx.sender);
    Ok(())
}
```

**Step 3: Verify it compiles**

Run: `cd packages/server/spacetimedb-module && cargo build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add packages/server/spacetimedb-module/
git commit -m "feat: scaffold SpacetimeDB server module with room table"
```

---

### Task 2: Define Room Management Tables & Reducers

**Files:**
- Modify: `packages/server/spacetimedb-module/src/lib.rs`

**Step 1: Add room_participant and room_asset tables**

```rust
#[spacetimedb::table(name = room_participant, public)]
pub struct RoomParticipant {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub room_id: String,
    pub identity: Identity,
    pub display_name: String,
    pub color: String,
    pub joined_at: Timestamp,
}

#[spacetimedb::table(name = room_asset, public)]
pub struct RoomAsset {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub room_id: String,
    pub asset_id: String,
    pub name: String,
    pub size_bytes: u64,
    pub mime_type: String,
    pub s3_url: Option<String>,
    pub uploaded_by: Identity,
}
```

**Step 2: Add create_room reducer**

```rust
fn generate_room_id() -> String {
    use rand::Rng;
    const CHARS: &[u8] = b"abcdefghijklmnopqrstuvwxyz0123456789";
    let mut rng = rand::thread_rng();
    (0..8).map(|_| CHARS[rng.gen_range(0..CHARS.len())] as char).collect()
}

#[reducer]
pub fn create_room(ctx: &ReducerContext) -> Result<(), String> {
    let id = generate_room_id();
    let now = Timestamp::now();
    ctx.db.room().insert(Room {
        id: id.clone(),
        creator_identity: ctx.sender,
        is_persistent: false,
        created_at: now,
        last_active_at: now,
    });
    log::info!("Room created: {}", id);
    Ok(())
}
```

**Step 3: Add join_room, leave_room, promote_room reducers**

```rust
const PALETTE: &[&str] = &["#FF6B6B", "#4ECDC4", "#45B7D1", "#96CEB4", "#FFEAA7", "#DDA0DD", "#98D8C8", "#F7DC6F"];

#[reducer]
pub fn join_room(ctx: &ReducerContext, room_id: String, display_name: String) -> Result<(), String> {
    let room = ctx.db.room().id().find(&room_id)
        .ok_or_else(|| format!("Room '{}' not found", room_id))?;
    let participant_count = ctx.db.room_participant().iter()
        .filter(|rp| rp.room_id == room_id)
        .count();
    let color = PALETTE[participant_count % PALETTE.len()].to_string();
    ctx.db.room_participant().insert(RoomParticipant {
        id: 0,
        room_id: room_id.clone(),
        identity: ctx.sender,
        display_name,
        color,
        joined_at: Timestamp::now(),
    });
    let mut room = room;
    room.last_active_at = Timestamp::now();
    ctx.db.room().id().update(room);
    Ok(())
}

#[reducer]
pub fn leave_room(ctx: &ReducerContext, room_id: String) -> Result<(), String> {
    let participant = ctx.db.room_participant().iter()
        .find(|rp| rp.room_id == room_id && rp.identity == ctx.sender)
        .ok_or("Not in this room")?;
    ctx.db.room_participant().id().delete(&participant.id);
    Ok(())
}

#[reducer]
pub fn promote_room(ctx: &ReducerContext, room_id: String) -> Result<(), String> {
    let mut room = ctx.db.room().id().find(&room_id)
        .ok_or_else(|| format!("Room '{}' not found", room_id))?;
    if room.creator_identity != ctx.sender {
        return Err("Only the creator can promote a room".to_string());
    }
    room.is_persistent = true;
    ctx.db.room().id().update(room);
    Ok(())
}
```

**Step 4: Add asset registration reducers**

```rust
#[reducer]
pub fn register_asset(
    ctx: &ReducerContext,
    room_id: String,
    asset_id: String,
    name: String,
    size_bytes: u64,
    mime_type: String,
) -> Result<(), String> {
    ctx.db.room().id().find(&room_id)
        .ok_or_else(|| format!("Room '{}' not found", room_id))?;
    if size_bytes > 100 * 1024 * 1024 {
        return Err("Asset too large (max 100MB)".to_string());
    }
    ctx.db.room_asset().insert(RoomAsset {
        id: 0,
        room_id,
        asset_id,
        name,
        size_bytes,
        mime_type,
        s3_url: None,
        uploaded_by: ctx.sender,
    });
    Ok(())
}

#[reducer]
pub fn update_asset_s3_url(
    ctx: &ReducerContext,
    room_id: String,
    asset_id: String,
    s3_url: String,
) -> Result<(), String> {
    let mut asset = ctx.db.room_asset().iter()
        .find(|ra| ra.room_id == room_id && ra.asset_id == asset_id)
        .ok_or("Asset not found")?;
    asset.s3_url = Some(s3_url);
    ctx.db.room_asset().id().update(asset);
    Ok(())
}
```

**Step 5: Verify it compiles**

Run: `cd packages/server/spacetimedb-module && cargo build`
Expected: Build succeeds

**Step 6: Commit**

```bash
git add packages/server/spacetimedb-module/
git commit -m "feat: add room management tables and reducers"
```

---

### Task 3: Add WebRTC Signaling & Presence Tables

**Files:**
- Modify: `packages/server/spacetimedb-module/src/lib.rs`

**Step 1: Add webrtc_signal table and reducer**

```rust
#[spacetimedb::table(name = webrtc_signal, public)]
pub struct WebrtcSignal {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub room_id: String,
    pub from_identity: Identity,
    pub to_identity: Identity,
    pub signal_type: String,
    pub payload: String,
    pub created_at: Timestamp,
}

#[reducer]
pub fn send_signal(
    ctx: &ReducerContext,
    room_id: String,
    to_identity: Identity,
    signal_type: String,
    payload: String,
) -> Result<(), String> {
    if !["offer", "answer", "ice"].contains(&signal_type.as_str()) {
        return Err(format!("Invalid signal type: {}", signal_type));
    }
    if payload.len() > 16 * 1024 {
        return Err("Signal payload too large (max 16KB)".to_string());
    }
    ctx.db.webrtc_signal().insert(WebrtcSignal {
        id: 0,
        room_id,
        from_identity: ctx.sender,
        to_identity,
        signal_type,
        payload,
        created_at: Timestamp::now(),
    });
    Ok(())
}
```

**Step 2: Add presence table and update reducer**

```rust
#[spacetimedb::table(name = presence, public)]
pub struct Presence {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub room_id: String,
    pub identity: Identity,
    pub display_name: String,
    pub color: String,
    pub cursor_x: f64,
    pub cursor_y: f64,
    pub cursor_target: String,
    pub last_seen: Timestamp,
}

#[reducer]
pub fn update_presence(
    ctx: &ReducerContext,
    room_id: String,
    cursor_x: f64,
    cursor_y: f64,
    cursor_target: String,
) -> Result<(), String> {
    let existing = ctx.db.presence().iter()
        .find(|p| p.room_id == room_id && p.identity == ctx.sender);
    match existing {
        Some(mut p) => {
            p.cursor_x = cursor_x;
            p.cursor_y = cursor_y;
            p.cursor_target = cursor_target;
            p.last_seen = Timestamp::now();
            ctx.db.presence().id().update(p);
        }
        None => {
            let participant = ctx.db.room_participant().iter()
                .find(|rp| rp.room_id == room_id && rp.identity == ctx.sender)
                .ok_or("Not in this room")?;
            ctx.db.presence().insert(Presence {
                id: 0,
                room_id,
                identity: ctx.sender,
                display_name: participant.display_name.clone(),
                color: participant.color.clone(),
                cursor_x,
                cursor_y,
                cursor_target,
                last_seen: Timestamp::now(),
            });
        }
    }
    Ok(())
}
```

**Step 3: Add BoxGraph sync tables**

```rust
#[spacetimedb::table(name = box_state, public)]
pub struct BoxState {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub room_id: String,
    pub box_uuid: String,
    pub box_name: String,
    pub data: String,
}

#[reducer]
pub fn box_create(
    ctx: &ReducerContext,
    room_id: String,
    box_uuid: String,
    box_name: String,
    data: String,
) -> Result<(), String> {
    if data.len() > 1024 * 1024 {
        return Err("Box data too large (max 1MB)".to_string());
    }
    ctx.db.box_state().insert(BoxState {
        id: 0,
        room_id,
        box_uuid,
        box_name,
        data,
    });
    Ok(())
}

#[reducer]
pub fn box_update(
    ctx: &ReducerContext,
    room_id: String,
    box_uuid: String,
    data: String,
) -> Result<(), String> {
    if data.len() > 1024 * 1024 {
        return Err("Box data too large (max 1MB)".to_string());
    }
    let mut state = ctx.db.box_state().iter()
        .find(|bs| bs.room_id == room_id && bs.box_uuid == box_uuid)
        .ok_or("Box not found")?;
    state.data = data;
    ctx.db.box_state().id().update(state);
    Ok(())
}

#[reducer]
pub fn box_delete(
    ctx: &ReducerContext,
    room_id: String,
    box_uuid: String,
) -> Result<(), String> {
    let state = ctx.db.box_state().iter()
        .find(|bs| bs.room_id == room_id && bs.box_uuid == box_uuid)
        .ok_or("Box not found")?;
    ctx.db.box_state().id().delete(&state.id);
    Ok(())
}
```

**Step 4: Add scheduled cleanup reducer**

```rust
#[spacetimedb::table(name = cleanup_schedule, scheduled(cleanup_stale_rooms))]
pub struct CleanupSchedule {
    #[primary_key]
    #[auto_inc]
    pub id: u64,
    pub scheduled_at: spacetimedb::ScheduleAt,
}

#[reducer]
pub fn cleanup_stale_rooms(ctx: &ReducerContext, _schedule: CleanupSchedule) -> Result<(), String> {
    let thirty_min_ago = Timestamp::now(); // TODO: subtract 30 minutes
    let stale_rooms: Vec<String> = ctx.db.room().iter()
        .filter(|room| !room.is_persistent)
        .filter(|room| {
            ctx.db.room_participant().iter()
                .filter(|rp| rp.room_id == room.id)
                .count() == 0
        })
        .map(|room| room.id.clone())
        .collect();
    for room_id in stale_rooms {
        for asset in ctx.db.room_asset().iter().filter(|ra| ra.room_id == room_id) {
            ctx.db.room_asset().id().delete(&asset.id);
        }
        for signal in ctx.db.webrtc_signal().iter().filter(|ws| ws.room_id == room_id) {
            ctx.db.webrtc_signal().id().delete(&signal.id);
        }
        for presence in ctx.db.presence().iter().filter(|p| p.room_id == room_id) {
            ctx.db.presence().id().delete(&presence.id);
        }
        for bs in ctx.db.box_state().iter().filter(|bs| bs.room_id == room_id) {
            ctx.db.box_state().id().delete(&bs.id);
        }
        ctx.db.room().id().delete(&room_id);
        log::info!("Cleaned up stale room: {}", room_id);
    }
    Ok(())
}
```

**Step 5: Verify it compiles**

Run: `cd packages/server/spacetimedb-module && cargo build`
Expected: Build succeeds

**Step 6: Commit**

```bash
git add packages/server/spacetimedb-module/
git commit -m "feat: add WebRTC signaling, presence, and box sync tables"
```

---

## Phase 2: TypeScript Client — Room & Sync Services

### Task 4: Create RoomService

**Files:**
- Create: `packages/studio/core/src/collab/RoomService.ts`
- Create: `packages/studio/core/src/collab/types.ts`
- Create: `packages/studio/core/src/collab/index.ts`
- Test: `packages/studio/core/src/collab/RoomService.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi} from "vitest"
import {RoomService} from "./RoomService"

describe("RoomService", () => {
    it("generates a share URL from a room ID", () => {
        const service = new RoomService({endpoint: "wss://localhost:3000"})
        const url = service.getShareUrl("abc12345")
        expect(url).toBe("https://opendaw.studio/r/abc12345")
    })
    it("extracts room ID from a share URL", () => {
        const roomId = RoomService.parseShareUrl("https://opendaw.studio/r/abc12345")
        expect(roomId).toBe("abc12345")
    })
    it("returns undefined for invalid share URLs", () => {
        const roomId = RoomService.parseShareUrl("https://opendaw.studio/other")
        expect(roomId).toBeUndefined()
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/RoomService.test.ts`
Expected: FAIL — module not found

**Step 3: Create types.ts**

```typescript
import {Optional} from "@opendaw/lib-std"

export type CollabConfig = {
    readonly endpoint: string
    readonly shareBaseUrl?: string
}

export type RoomInfo = {
    readonly id: string
    readonly isPersistent: boolean
    readonly participantCount: number
}

export type Participant = {
    readonly identity: string
    readonly displayName: string
    readonly color: string
}

export type AssetMeta = {
    readonly assetId: string
    readonly name: string
    readonly sizeBytes: number
    readonly mimeType: string
    readonly s3Url: Optional<string>
}

export type PresenceData = {
    readonly identity: string
    readonly displayName: string
    readonly color: string
    readonly cursorX: number
    readonly cursorY: number
    readonly cursorTarget: string
}

export type S3Config = {
    readonly bucket: string
    readonly region: string
    readonly accessKeyId: string
    readonly secretAccessKey: string
    readonly endpoint?: string
}
```

**Step 4: Create RoomService.ts**

```typescript
import {Optional} from "@opendaw/lib-std"
import {CollabConfig} from "./types"

const DEFAULT_SHARE_BASE = "https://opendaw.studio"

export class RoomService {
    readonly #config: CollabConfig
    readonly #shareBase: string

    constructor(config: CollabConfig) {
        this.#config = config
        this.#shareBase = config.shareBaseUrl ?? DEFAULT_SHARE_BASE
    }

    getShareUrl(roomId: string): string {
        return `${this.#shareBase}/r/${roomId}`
    }

    static parseShareUrl(url: string): Optional<string> {
        const match = url.match(/\/r\/([a-z0-9]+)$/)
        return match?.[1]
    }

    get endpoint(): string {
        return this.#config.endpoint
    }
}
```

**Step 5: Create index.ts**

```typescript
export {RoomService} from "./RoomService"
export type {CollabConfig, RoomInfo, Participant, AssetMeta, PresenceData, S3Config} from "./types"
```

**Step 6: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/RoomService.test.ts`
Expected: PASS

**Step 7: Commit**

```bash
git add packages/studio/core/src/collab/
git commit -m "feat: add RoomService with URL generation and parsing"
```

---

### Task 5: Create AssetTransport Interface & OPFS Source

**Files:**
- Create: `packages/studio/core/src/collab/assets/AssetTransport.ts`
- Create: `packages/studio/core/src/collab/assets/OpfsAssetSource.ts`
- Create: `packages/studio/core/src/collab/assets/index.ts`
- Test: `packages/studio/core/src/collab/assets/AssetTransport.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi} from "vitest"
import {AssetTransportChain} from "./AssetTransport"
import {AssetSource} from "./AssetTransport"

describe("AssetTransportChain", () => {
    const createMockSource = (name: string, assets: Map<string, ArrayBuffer>): AssetSource => ({
        name,
        resolve: vi.fn(async (assetId: string) => assets.get(assetId)),
        publish: vi.fn(async () => {}),
    })
    it("resolves from the first source that has the asset", async () => {
        const data = new ArrayBuffer(8)
        const source1 = createMockSource("empty", new Map())
        const source2 = createMockSource("has-it", new Map([["asset-1", data]]))
        const chain = new AssetTransportChain([source1, source2])
        const result = await chain.resolve("asset-1")
        expect(result).toBe(data)
        expect(source1.resolve).toHaveBeenCalledWith("asset-1")
        expect(source2.resolve).toHaveBeenCalledWith("asset-1")
    })
    it("returns undefined when no source has the asset", async () => {
        const source1 = createMockSource("empty1", new Map())
        const source2 = createMockSource("empty2", new Map())
        const chain = new AssetTransportChain([source1, source2])
        const result = await chain.resolve("missing")
        expect(result).toBeUndefined()
    })
    it("publishes to all sources", async () => {
        const source1 = createMockSource("s1", new Map())
        const source2 = createMockSource("s2", new Map())
        const chain = new AssetTransportChain([source1, source2])
        const data = new ArrayBuffer(4)
        const meta = {assetId: "a1", name: "test.wav", sizeBytes: 4, mimeType: "audio/wav", s3Url: undefined}
        await chain.publish("a1", data, meta)
        expect(source1.publish).toHaveBeenCalledWith("a1", data, meta)
        expect(source2.publish).toHaveBeenCalledWith("a1", data, meta)
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/AssetTransport.test.ts`
Expected: FAIL — module not found

**Step 3: Create AssetTransport.ts**

```typescript
import {Optional} from "@opendaw/lib-std"
import {AssetMeta} from "../types"

export interface AssetSource {
    readonly name: string
    resolve(assetId: string): Promise<Optional<ArrayBuffer>>
    publish(assetId: string, data: ArrayBuffer, meta: AssetMeta): Promise<void>
}

export class AssetTransportChain {
    readonly #sources: ReadonlyArray<AssetSource>

    constructor(sources: ReadonlyArray<AssetSource>) {
        this.#sources = sources
    }

    async resolve(assetId: string): Promise<Optional<ArrayBuffer>> {
        for (const source of this.#sources) {
            const result = await source.resolve(assetId)
            if (result !== undefined) {
                console.debug(`Asset '${assetId}' resolved from ${source.name}`)
                return result
            }
        }
        console.warn(`Asset '${assetId}' not found in any source`)
        return undefined
    }

    async publish(assetId: string, data: ArrayBuffer, meta: AssetMeta): Promise<void> {
        await Promise.all(this.#sources.map(source => source.publish(assetId, data, meta)))
    }
}
```

**Step 4: Create OpfsAssetSource.ts**

```typescript
import {Optional} from "@opendaw/lib-std"
import {AssetSource} from "./AssetTransport"
import {AssetMeta} from "../types"

export class OpfsAssetSource implements AssetSource {
    readonly name = "opfs"
    readonly #folder: string

    constructor(folder: string = "collab-assets") {
        this.#folder = folder
    }

    async resolve(assetId: string): Promise<Optional<ArrayBuffer>> {
        try {
            const root = await navigator.storage.getDirectory()
            const dir = await root.getDirectoryHandle(this.#folder)
            const file = await dir.getFileHandle(assetId)
            const blob = await file.getFile()
            return await blob.arrayBuffer()
        } catch {
            return undefined
        }
    }

    async publish(assetId: string, data: ArrayBuffer, _meta: AssetMeta): Promise<void> {
        const root = await navigator.storage.getDirectory()
        const dir = await root.getDirectoryHandle(this.#folder, {create: true})
        const file = await dir.getFileHandle(assetId, {create: true})
        const writable = await file.createWritable()
        await writable.write(data)
        await writable.close()
    }
}
```

**Step 5: Create index.ts**

```typescript
export {AssetTransportChain} from "./AssetTransport"
export type {AssetSource} from "./AssetTransport"
export {OpfsAssetSource} from "./OpfsAssetSource"
```

**Step 6: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/AssetTransport.test.ts`
Expected: PASS

**Step 7: Commit**

```bash
git add packages/studio/core/src/collab/assets/
git commit -m "feat: add AssetTransportChain with OPFS source"
```

---

### Task 6: Create S3 Asset Source

**Files:**
- Create: `packages/studio/core/src/collab/assets/S3AssetSource.ts`
- Test: `packages/studio/core/src/collab/assets/S3AssetSource.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi} from "vitest"
import {S3AssetSource} from "./S3AssetSource"
import {S3Config} from "../types"

describe("S3AssetSource", () => {
    const config: S3Config = {
        bucket: "my-bucket",
        region: "us-east-1",
        accessKeyId: "AKIA...",
        secretAccessKey: "secret",
    }
    it("constructs the correct S3 URL for an asset", () => {
        const source = new S3AssetSource(config)
        expect(source.getUrl("asset-123")).toBe(
            "https://my-bucket.s3.us-east-1.amazonaws.com/opendaw/assets/asset-123"
        )
    })
    it("uses custom endpoint when provided", () => {
        const customConfig = {...config, endpoint: "https://minio.local:9000"}
        const source = new S3AssetSource(customConfig)
        expect(source.getUrl("asset-123")).toBe(
            "https://minio.local:9000/my-bucket/opendaw/assets/asset-123"
        )
    })
    it("has name 's3'", () => {
        const source = new S3AssetSource(config)
        expect(source.name).toBe("s3")
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/S3AssetSource.test.ts`
Expected: FAIL — module not found

**Step 3: Create S3AssetSource.ts**

```typescript
import {isDefined, Optional} from "@opendaw/lib-std"
import {AssetSource} from "./AssetTransport"
import {AssetMeta, S3Config} from "../types"

const ASSET_PREFIX = "opendaw/assets"

export class S3AssetSource implements AssetSource {
    readonly name = "s3"
    readonly #config: S3Config

    constructor(config: S3Config) {
        this.#config = config
    }

    getUrl(assetId: string): string {
        if (isDefined(this.#config.endpoint)) {
            return `${this.#config.endpoint}/${this.#config.bucket}/${ASSET_PREFIX}/${assetId}`
        }
        return `https://${this.#config.bucket}.s3.${this.#config.region}.amazonaws.com/${ASSET_PREFIX}/${assetId}`
    }

    async resolve(assetId: string): Promise<Optional<ArrayBuffer>> {
        try {
            const response = await fetch(this.getUrl(assetId))
            if (!response.ok) {return undefined}
            return await response.arrayBuffer()
        } catch {
            return undefined
        }
    }

    async publish(assetId: string, data: ArrayBuffer, _meta: AssetMeta): Promise<void> {
        const url = this.getUrl(assetId)
        const response = await fetch(url, {
            method: "PUT",
            body: data,
            headers: {"Content-Type": "application/octet-stream"},
        })
        if (!response.ok) {
            throw new Error(`S3 upload failed: ${response.status} ${response.statusText}`)
        }
    }
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/S3AssetSource.test.ts`
Expected: PASS

**Step 5: Update index.ts and commit**

```bash
git add packages/studio/core/src/collab/assets/
git commit -m "feat: add S3AssetSource with URL construction and fetch-based upload"
```

---

### Task 7: Create WebRTC Peer Connection Manager

**Files:**
- Create: `packages/studio/core/src/collab/webrtc/PeerManager.ts`
- Create: `packages/studio/core/src/collab/webrtc/types.ts`
- Create: `packages/studio/core/src/collab/webrtc/index.ts`
- Test: `packages/studio/core/src/collab/webrtc/PeerManager.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi, beforeEach} from "vitest"
import {PeerManager} from "./PeerManager"

describe("PeerManager", () => {
    it("tracks connected peer IDs", () => {
        const manager = new PeerManager()
        expect(manager.peerIds).toEqual([])
    })
    it("emits onPeerConnected when a peer connects", () => {
        const manager = new PeerManager()
        const spy = vi.fn()
        manager.onPeerConnected.subscribe(spy)
        manager.addPeer("peer-1")
        expect(spy).toHaveBeenCalledWith("peer-1")
    })
    it("emits onPeerDisconnected when a peer disconnects", () => {
        const manager = new PeerManager()
        const spy = vi.fn()
        manager.onPeerDisconnected.subscribe(spy)
        manager.addPeer("peer-1")
        manager.removePeer("peer-1")
        expect(spy).toHaveBeenCalledWith("peer-1")
    })
    it("does not add duplicate peers", () => {
        const manager = new PeerManager()
        const spy = vi.fn()
        manager.onPeerConnected.subscribe(spy)
        manager.addPeer("peer-1")
        manager.addPeer("peer-1")
        expect(spy).toHaveBeenCalledTimes(1)
        expect(manager.peerIds).toEqual(["peer-1"])
    })
    it("cleans up all peers on terminate", () => {
        const manager = new PeerManager()
        manager.addPeer("peer-1")
        manager.addPeer("peer-2")
        manager.terminate()
        expect(manager.peerIds).toEqual([])
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/webrtc/PeerManager.test.ts`
Expected: FAIL — module not found

**Step 3: Create types.ts**

```typescript
export type SignalMessage = {
    readonly fromIdentity: string
    readonly toIdentity: string
    readonly signalType: "offer" | "answer" | "ice"
    readonly payload: string
}

export type AssetRequest = {
    readonly type: "request"
    readonly assetId: string
}

export type AssetResponse = {
    readonly type: "response"
    readonly assetId: string
    readonly data: ArrayBuffer
}

export type AssetNotFound = {
    readonly type: "not-found"
    readonly assetId: string
}

export type DataChannelMessage = AssetRequest | AssetResponse | AssetNotFound
```

**Step 4: Create PeerManager.ts**

```typescript
import {Notifier, Terminable} from "@opendaw/lib-std"

export class PeerManager implements Terminable {
    readonly #peers: Map<string, RTCPeerConnection> = new Map()
    readonly onPeerConnected: Notifier<string> = new Notifier()
    readonly onPeerDisconnected: Notifier<string> = new Notifier()

    get peerIds(): ReadonlyArray<string> {
        return Array.from(this.#peers.keys())
    }

    addPeer(peerId: string): void {
        if (this.#peers.has(peerId)) {return}
        this.#peers.set(peerId, new RTCPeerConnection())
        this.onPeerConnected.notify(peerId)
    }

    removePeer(peerId: string): void {
        const connection = this.#peers.get(peerId)
        if (connection === undefined) {return}
        connection.close()
        this.#peers.delete(peerId)
        this.onPeerDisconnected.notify(peerId)
    }

    getConnection(peerId: string): RTCPeerConnection | undefined {
        return this.#peers.get(peerId)
    }

    terminate(): void {
        for (const [peerId, connection] of this.#peers) {
            connection.close()
            this.onPeerDisconnected.notify(peerId)
        }
        this.#peers.clear()
        this.onPeerConnected.terminate()
        this.onPeerDisconnected.terminate()
    }
}
```

**Step 5: Create index.ts**

```typescript
export {PeerManager} from "./PeerManager"
export type {SignalMessage, AssetRequest, AssetResponse, AssetNotFound, DataChannelMessage} from "./types"
```

**Step 6: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/webrtc/PeerManager.test.ts`
Expected: PASS

**Step 7: Commit**

```bash
git add packages/studio/core/src/collab/webrtc/
git commit -m "feat: add PeerManager for WebRTC connection tracking"
```

---

### Task 8: Create WebRTC Asset Source

**Files:**
- Create: `packages/studio/core/src/collab/assets/WebRTCAssetSource.ts`
- Test: `packages/studio/core/src/collab/assets/WebRTCAssetSource.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi} from "vitest"
import {WebRTCAssetSource} from "./WebRTCAssetSource"
import {PeerManager} from "../webrtc/PeerManager"

describe("WebRTCAssetSource", () => {
    it("has name 'webrtc'", () => {
        const peerManager = new PeerManager()
        const source = new WebRTCAssetSource(peerManager)
        expect(source.name).toBe("webrtc")
    })
    it("returns undefined when no peers are connected", async () => {
        const peerManager = new PeerManager()
        const source = new WebRTCAssetSource(peerManager)
        const result = await source.resolve("asset-1")
        expect(result).toBeUndefined()
    })
    it("publish is a no-op (assets are pulled, not pushed)", async () => {
        const peerManager = new PeerManager()
        const source = new WebRTCAssetSource(peerManager)
        await expect(source.publish("a", new ArrayBuffer(0), {
            assetId: "a", name: "test", sizeBytes: 0, mimeType: "audio/wav", s3Url: undefined
        })).resolves.toBeUndefined()
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/WebRTCAssetSource.test.ts`
Expected: FAIL — module not found

**Step 3: Create WebRTCAssetSource.ts**

```typescript
import {Optional} from "@opendaw/lib-std"
import {AssetSource} from "./AssetTransport"
import {AssetMeta} from "../types"
import {PeerManager} from "../webrtc/PeerManager"

export class WebRTCAssetSource implements AssetSource {
    readonly name = "webrtc"
    readonly #peerManager: PeerManager

    constructor(peerManager: PeerManager) {
        this.#peerManager = peerManager
    }

    async resolve(assetId: string): Promise<Optional<ArrayBuffer>> {
        const peerIds = this.#peerManager.peerIds
        if (peerIds.length === 0) {return undefined}
        for (const peerId of peerIds) {
            const result = await this.#requestFromPeer(peerId, assetId)
            if (result !== undefined) {return result}
        }
        return undefined
    }

    async publish(_assetId: string, _data: ArrayBuffer, _meta: AssetMeta): Promise<void> {
        // WebRTC assets are pulled by requesters, not pushed
    }

    async #requestFromPeer(_peerId: string, _assetId: string): Promise<Optional<ArrayBuffer>> {
        // TODO: Implement WebRTC data channel request/response protocol
        // 1. Get data channel from PeerManager
        // 2. Send AssetRequest message
        // 3. Wait for AssetResponse or AssetNotFound with timeout
        return undefined
    }
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/assets/WebRTCAssetSource.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add packages/studio/core/src/collab/assets/
git commit -m "feat: add WebRTCAssetSource with pull-based asset resolution"
```

---

### Task 9: Create PresenceService

**Files:**
- Create: `packages/studio/core/src/collab/PresenceService.ts`
- Test: `packages/studio/core/src/collab/PresenceService.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi, beforeEach} from "vitest"
import {PresenceService} from "./PresenceService"

describe("PresenceService", () => {
    it("starts with empty participants", () => {
        const service = new PresenceService()
        expect(service.participants).toEqual([])
    })
    it("adds a participant and notifies", () => {
        const service = new PresenceService()
        const spy = vi.fn()
        service.onChange.subscribe(spy)
        service.updatePresence({
            identity: "user-1",
            displayName: "Alice",
            color: "#FF6B6B",
            cursorX: 100,
            cursorY: 200,
            cursorTarget: "track-1"
        })
        expect(service.participants).toHaveLength(1)
        expect(service.participants[0].displayName).toBe("Alice")
        expect(spy).toHaveBeenCalled()
    })
    it("updates existing participant position", () => {
        const service = new PresenceService()
        service.updatePresence({
            identity: "user-1", displayName: "Alice", color: "#FF6B6B",
            cursorX: 100, cursorY: 200, cursorTarget: "track-1"
        })
        service.updatePresence({
            identity: "user-1", displayName: "Alice", color: "#FF6B6B",
            cursorX: 300, cursorY: 400, cursorTarget: "track-2"
        })
        expect(service.participants).toHaveLength(1)
        expect(service.participants[0].cursorX).toBe(300)
    })
    it("removes a participant", () => {
        const service = new PresenceService()
        service.updatePresence({
            identity: "user-1", displayName: "Alice", color: "#FF6B6B",
            cursorX: 0, cursorY: 0, cursorTarget: ""
        })
        service.removeParticipant("user-1")
        expect(service.participants).toEqual([])
    })
    it("cleans up on terminate", () => {
        const service = new PresenceService()
        service.updatePresence({
            identity: "user-1", displayName: "Alice", color: "#FF6B6B",
            cursorX: 0, cursorY: 0, cursorTarget: ""
        })
        service.terminate()
        expect(service.participants).toEqual([])
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/PresenceService.test.ts`
Expected: FAIL — module not found

**Step 3: Create PresenceService.ts**

```typescript
import {Notifier, Terminable} from "@opendaw/lib-std"
import {PresenceData} from "./types"

export class PresenceService implements Terminable {
    readonly #participants: Map<string, PresenceData> = new Map()
    readonly onChange: Notifier<void> = new Notifier()

    get participants(): ReadonlyArray<PresenceData> {
        return Array.from(this.#participants.values())
    }

    updatePresence(data: PresenceData): void {
        this.#participants.set(data.identity, data)
        this.onChange.notify()
    }

    removeParticipant(identity: string): void {
        if (!this.#participants.has(identity)) {return}
        this.#participants.delete(identity)
        this.onChange.notify()
    }

    terminate(): void {
        this.#participants.clear()
        this.onChange.terminate()
    }
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/PresenceService.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add packages/studio/core/src/collab/PresenceService.ts packages/studio/core/src/collab/PresenceService.test.ts
git commit -m "feat: add PresenceService for tracking collaborator cursors"
```

---

### Task 10: Create CollabService (Orchestrator)

**Files:**
- Create: `packages/studio/core/src/collab/CollabService.ts`
- Test: `packages/studio/core/src/collab/CollabService.test.ts`

**Step 1: Write the failing test**

```typescript
import {describe, expect, it, vi} from "vitest"
import {CollabService, CollabState} from "./CollabService"

describe("CollabService", () => {
    it("starts in disconnected state", () => {
        const service = new CollabService({endpoint: "wss://localhost:3000"})
        expect(service.state).toBe(CollabState.Disconnected)
    })
    it("exposes room service", () => {
        const service = new CollabService({endpoint: "wss://localhost:3000"})
        expect(service.room).toBeDefined()
    })
    it("exposes presence service", () => {
        const service = new CollabService({endpoint: "wss://localhost:3000"})
        expect(service.presence).toBeDefined()
    })
    it("cleans up on terminate", () => {
        const service = new CollabService({endpoint: "wss://localhost:3000"})
        service.terminate()
        expect(service.state).toBe(CollabState.Disconnected)
    })
})
```

**Step 2: Run test to verify it fails**

Run: `cd packages/studio/core && npx vitest run src/collab/CollabService.test.ts`
Expected: FAIL — module not found

**Step 3: Create CollabService.ts**

```typescript
import {Terminable, Terminator} from "@opendaw/lib-std"
import {CollabConfig} from "./types"
import {RoomService} from "./RoomService"
import {PresenceService} from "./PresenceService"
import {PeerManager} from "./webrtc/PeerManager"
import {AssetTransportChain} from "./assets/AssetTransport"
import {OpfsAssetSource} from "./assets/OpfsAssetSource"
import {WebRTCAssetSource} from "./assets/WebRTCAssetSource"

export enum CollabState {
    Disconnected = "disconnected",
    Connecting = "connecting",
    Connected = "connected",
}

export class CollabService implements Terminable {
    readonly #terminator = new Terminator()
    readonly #config: CollabConfig
    readonly room: RoomService
    readonly presence: PresenceService
    readonly peerManager: PeerManager
    readonly assets: AssetTransportChain
    #state: CollabState = CollabState.Disconnected

    constructor(config: CollabConfig) {
        this.#config = config
        this.room = new RoomService(config)
        this.presence = new PresenceService()
        this.peerManager = new PeerManager()
        this.assets = new AssetTransportChain([
            new OpfsAssetSource(),
            new WebRTCAssetSource(this.peerManager),
        ])
        this.#terminator.ownAll(this.presence, this.peerManager)
    }

    get state(): CollabState {
        return this.#state
    }

    terminate(): void {
        this.#state = CollabState.Disconnected
        this.#terminator.terminate()
    }
}
```

**Step 4: Run test to verify it passes**

Run: `cd packages/studio/core && npx vitest run src/collab/CollabService.test.ts`
Expected: PASS

**Step 5: Update collab/index.ts to export all**

```typescript
export {RoomService} from "./RoomService"
export {PresenceService} from "./PresenceService"
export {CollabService, CollabState} from "./CollabService"
export type {CollabConfig, RoomInfo, Participant, AssetMeta, PresenceData, S3Config} from "./types"
```

**Step 6: Commit**

```bash
git add packages/studio/core/src/collab/
git commit -m "feat: add CollabService orchestrator with asset transport chain"
```

---

## Phase 3: Collaboration UI

### Task 11: Create Share Dialog Component

**Files:**
- Create: `packages/app/studio/src/ui/collab/ShareDialog.tsx`
- Create: `packages/app/studio/src/ui/collab/index.ts`

**Step 1: Create ShareDialog.tsx**

```tsx
import {useCallback, useState} from "react"

type ShareDialogProps = {
    readonly shareUrl: string
    readonly onClose: () => void
    readonly onPromoteRoom: () => void
    readonly isPersistent: boolean
}

export const ShareDialog = ({shareUrl, onClose, onPromoteRoom, isPersistent}: ShareDialogProps) => {
    const [copied, setCopied] = useState(false)
    const handleCopy = useCallback(async () => {
        await navigator.clipboard.writeText(shareUrl)
        setCopied(true)
        setTimeout(() => setCopied(false), 2000)
    }, [shareUrl])
    return (
        <div className="share-dialog">
            <h3>Share this session</h3>
            <div className="share-url-row">
                <input type="text" value={shareUrl} readOnly />
                <button onClick={handleCopy}>{copied ? "Copied!" : "Copy Link"}</button>
            </div>
            <p className="share-hint">
                Anyone with this link can join and collaborate in real-time.
            </p>
            {!isPersistent && (
                <div className="share-persist">
                    <p>This session is temporary and will expire after everyone leaves.</p>
                    <button onClick={onPromoteRoom}>Make Persistent</button>
                </div>
            )}
            <button className="share-close" onClick={onClose}>Close</button>
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/
git commit -m "feat: add ShareDialog component"
```

---

### Task 12: Create Join Screen Component

**Files:**
- Create: `packages/app/studio/src/ui/collab/JoinScreen.tsx`

**Step 1: Create JoinScreen.tsx**

```tsx
import {useCallback, useState} from "react"

type JoinScreenProps = {
    readonly roomId: string
    readonly onJoin: (displayName: string) => void
    readonly onCancel: () => void
    readonly isConnecting: boolean
}

export const JoinScreen = ({roomId, onJoin, onCancel, isConnecting}: JoinScreenProps) => {
    const savedName = localStorage.getItem("opendaw-display-name") ?? ""
    const [displayName, setDisplayName] = useState(savedName)
    const handleJoin = useCallback(() => {
        const name = displayName.trim()
        if (name.length === 0) {return}
        localStorage.setItem("opendaw-display-name", name)
        onJoin(name)
    }, [displayName, onJoin])
    return (
        <div className="join-screen">
            <h2>Join Session</h2>
            <p>Room: <code>{roomId}</code></p>
            <label>
                Your name
                <input
                    type="text"
                    value={displayName}
                    onChange={event => setDisplayName(event.target.value)}
                    placeholder="Enter your name"
                    maxLength={32}
                    autoFocus
                    onKeyDown={event => event.key === "Enter" && handleJoin()}
                />
            </label>
            <div className="join-actions">
                <button onClick={handleJoin} disabled={isConnecting || displayName.trim().length === 0}>
                    {isConnecting ? "Connecting..." : "Join"}
                </button>
                <button onClick={onCancel}>Cancel</button>
            </div>
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/JoinScreen.tsx
git commit -m "feat: add JoinScreen component with name persistence"
```

---

### Task 13: Create Participant Panel Component

**Files:**
- Create: `packages/app/studio/src/ui/collab/ParticipantPanel.tsx`

**Step 1: Create ParticipantPanel.tsx**

```tsx
import {PresenceData} from "@opendaw/studio-core"

type ParticipantPanelProps = {
    readonly participants: ReadonlyArray<PresenceData>
    readonly localIdentity: string
    readonly isOpen: boolean
    readonly onToggle: () => void
}

export const ParticipantPanel = ({participants, localIdentity, isOpen, onToggle}: ParticipantPanelProps) => {
    const totalCount = participants.length + 1
    return (
        <div className="participant-panel">
            <button className="participant-toggle" onClick={onToggle}>
                <span className="participant-count">{totalCount}</span>
                <span>collaborators</span>
            </button>
            {isOpen && (
                <div className="participant-list">
                    <div className="participant-item local">
                        <span className="participant-dot" style={{background: "#4ECDC4"}} />
                        <span>You</span>
                    </div>
                    {participants.map(participant => (
                        <div key={participant.identity} className="participant-item">
                            <span className="participant-dot" style={{background: participant.color}} />
                            <span>{participant.displayName}</span>
                        </div>
                    ))}
                </div>
            )}
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/ParticipantPanel.tsx
git commit -m "feat: add ParticipantPanel component"
```

---

### Task 14: Create Cursor Overlay Component

**Files:**
- Create: `packages/app/studio/src/ui/collab/CursorOverlay.tsx`

**Step 1: Create CursorOverlay.tsx**

```tsx
import {PresenceData} from "@opendaw/studio-core"

type CursorOverlayProps = {
    readonly participants: ReadonlyArray<PresenceData>
}

export const CursorOverlay = ({participants}: CursorOverlayProps) => {
    return (
        <div className="cursor-overlay" style={{position: "absolute", inset: 0, pointerEvents: "none", zIndex: 1000}}>
            {participants.map(participant => (
                <div
                    key={participant.identity}
                    className="remote-cursor"
                    style={{
                        position: "absolute",
                        left: participant.cursorX,
                        top: participant.cursorY,
                        transition: "left 0.1s, top 0.1s",
                    }}
                >
                    <svg width="16" height="20" viewBox="0 0 16 20" fill={participant.color}>
                        <path d="M0 0 L0 16 L4 12 L8 20 L10 19 L6 11 L12 11 Z" />
                    </svg>
                    <span
                        className="cursor-label"
                        style={{
                            background: participant.color,
                            color: "white",
                            fontSize: "11px",
                            padding: "1px 4px",
                            borderRadius: "3px",
                            whiteSpace: "nowrap",
                            position: "absolute",
                            left: "14px",
                            top: "12px",
                        }}
                    >
                        {participant.displayName}
                    </span>
                </div>
            ))}
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/CursorOverlay.tsx
git commit -m "feat: add CursorOverlay for remote participant cursors"
```

---

### Task 15: Create collab UI index and update exports

**Files:**
- Create: `packages/app/studio/src/ui/collab/index.ts`

**Step 1: Create index.ts**

```typescript
export {ShareDialog} from "./ShareDialog"
export {JoinScreen} from "./JoinScreen"
export {ParticipantPanel} from "./ParticipantPanel"
export {CursorOverlay} from "./CursorOverlay"
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/index.ts
git commit -m "feat: export collab UI components"
```

---

## Phase 4: Menu Integration & URL Routing

### Task 16: Add Collaboration Menu Items to StudioMenu

**Files:**
- Modify: `packages/app/studio/src/service/StudioMenu.ts:110-138`

**Step 1: Replace the YSync "Connect Room..." menu item with SpacetimeDB collaboration**

Replace the existing experimental features section (lines 110-138) with:

```typescript
MenuItem.default({
    label: "Collaborate",
    separatorBefore: true
}).setRuntimeChildrenProcedure(parent => {
    parent.addMenuItem(
        MenuItem.default({label: "Share Session..."})
            .setTriggerProcedure(async () => {
                // TODO: Wire to CollabService.createRoom() + show ShareDialog
                const dialog = RuntimeNotifier.progress({
                    headline: "Creating Session...",
                    message: "Setting up a collaborative session..."
                })
                dialog.terminate()
            }),
        MenuItem.default({label: "Join Session..."})
            .setTriggerProcedure(async () => {
                const url = prompt("Enter a session URL or room ID:", "")
                if (isAbsent(url)) {return}
                // TODO: Wire to CollabService.joinRoom() + show JoinScreen
            }),
        MenuItem.default({
            label: "Leave Session",
            selectable: false // Enable when in a session
        }).setTriggerProcedure(async () => {
            // TODO: Wire to CollabService.leaveRoom()
        })
    )
}),
MenuItem.default({
    label: "Experimental Features",
    hidden: !StudioPreferences.settings.debug["enable-beta-features"],
    separatorBefore: true
}).setRuntimeChildrenProcedure(parent => {
    parent.addMenuItem(
        MenuItem.default({label: "Connect Room (Legacy)..."})
            .setTriggerProcedure(async () => {
                const roomName = prompt("Enter a room name:", "")
                if (isAbsent(roomName)) {return}
                const dialog = RuntimeNotifier.progress({
                    headline: "Connecting to Room...",
                    message: "Please wait while we connect to the room..."
                })
                const {status, value: project, error} = await Promises.tryCatch(
                    YService.getOrCreateRoom(service.projectProfileService.getValue()
                        .map(profile => profile.project), service, roomName))
                if (status === "resolved") {
                    service.projectProfileService.setProject(project, roomName)
                } else {
                    await RuntimeNotifier.info({
                        headline: "Failed Connecting Room",
                        message: String(error)
                    })
                }
                dialog.terminate()
            })
    )
}),
```

**Step 2: Verify the project builds**

Run: `npx turbo build`
Expected: Build succeeds

**Step 3: Commit**

```bash
git add packages/app/studio/src/service/StudioMenu.ts
git commit -m "feat: add Collaborate menu with Share/Join/Leave items"
```

---

### Task 17: Add S3 Settings to Preferences

**Files:**
- Create: `packages/app/studio/src/ui/collab/S3Settings.tsx`

**Step 1: Create S3Settings.tsx**

```tsx
import {useCallback, useState} from "react"
import {S3Config} from "@opendaw/studio-core"

const STORAGE_KEY = "opendaw-s3-config"

const loadConfig = (): Partial<S3Config> => {
    try {
        const raw = localStorage.getItem(STORAGE_KEY)
        return raw ? JSON.parse(raw) : {}
    } catch {
        return {}
    }
}

const saveConfig = (config: S3Config): void => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(config))
}

export const S3Settings = () => {
    const [config, setConfig] = useState<Partial<S3Config>>(loadConfig)
    const [saved, setSaved] = useState(false)
    const handleSave = useCallback(() => {
        if (!config.bucket || !config.region || !config.accessKeyId || !config.secretAccessKey) {return}
        saveConfig(config as S3Config)
        setSaved(true)
        setTimeout(() => setSaved(false), 2000)
    }, [config])
    const handleClear = useCallback(() => {
        localStorage.removeItem(STORAGE_KEY)
        setConfig({})
    }, [])
    const update = (field: keyof S3Config, value: string) => {
        setConfig(prev => ({...prev, [field]: value}))
    }
    return (
        <div className="s3-settings">
            <h3>S3 Storage (Optional)</h3>
            <p>Configure your own S3-compatible storage for persistent asset hosting.</p>
            <label>Bucket<input type="text" value={config.bucket ?? ""} onChange={event => update("bucket", event.target.value)} /></label>
            <label>Region<input type="text" value={config.region ?? ""} onChange={event => update("region", event.target.value)} /></label>
            <label>Access Key ID<input type="text" value={config.accessKeyId ?? ""} onChange={event => update("accessKeyId", event.target.value)} /></label>
            <label>Secret Access Key<input type="password" value={config.secretAccessKey ?? ""} onChange={event => update("secretAccessKey", event.target.value)} /></label>
            <label>Custom Endpoint (optional)<input type="text" value={config.endpoint ?? ""} onChange={event => update("endpoint", event.target.value)} placeholder="https://minio.local:9000" /></label>
            <div className="s3-actions">
                <button onClick={handleSave}>{saved ? "Saved!" : "Save"}</button>
                <button onClick={handleClear}>Clear</button>
            </div>
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/S3Settings.tsx
git commit -m "feat: add S3Settings component for user-provided storage config"
```

---

### Task 18: Add Collab Settings to Preferences

**Files:**
- Create: `packages/app/studio/src/ui/collab/CollabSettings.tsx`

**Step 1: Create CollabSettings.tsx**

```tsx
import {useCallback, useState} from "react"

const ENDPOINT_KEY = "opendaw-stdb-endpoint"
const DEFAULT_ENDPOINT = "wss://live.opendaw.studio"

export const CollabSettings = () => {
    const [endpoint, setEndpoint] = useState(
        localStorage.getItem(ENDPOINT_KEY) ?? DEFAULT_ENDPOINT
    )
    const [saved, setSaved] = useState(false)
    const handleSave = useCallback(() => {
        localStorage.setItem(ENDPOINT_KEY, endpoint)
        setSaved(true)
        setTimeout(() => setSaved(false), 2000)
    }, [endpoint])
    const handleReset = useCallback(() => {
        localStorage.removeItem(ENDPOINT_KEY)
        setEndpoint(DEFAULT_ENDPOINT)
    }, [])
    return (
        <div className="collab-settings">
            <h3>Collaboration Server</h3>
            <label>
                SpacetimeDB Endpoint
                <input
                    type="text"
                    value={endpoint}
                    onChange={event => setEndpoint(event.target.value)}
                    placeholder={DEFAULT_ENDPOINT}
                />
            </label>
            <div className="collab-settings-actions">
                <button onClick={handleSave}>{saved ? "Saved!" : "Save"}</button>
                <button onClick={handleReset}>Reset to Default</button>
            </div>
        </div>
    )
}
```

**Step 2: Commit**

```bash
git add packages/app/studio/src/ui/collab/CollabSettings.tsx
git commit -m "feat: add CollabSettings for configurable SpacetimeDB endpoint"
```

---

## Phase 5: Integration & Wiring

### Task 19: Wire CollabService into StudioService

**Files:**
- Modify: `packages/app/studio/src/service/StudioService.ts`
- Modify: `packages/studio/core/src/collab/index.ts`

This task connects the CollabService to the main application service so the menu items and UI components can access it. The exact modifications depend on how StudioService is structured — read the file first and add CollabService as a property.

**Step 1: Read StudioService.ts to understand the pattern**

**Step 2: Add CollabService as a property of StudioService**

**Step 3: Verify build succeeds**

Run: `npx turbo build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add packages/app/studio/src/service/StudioService.ts packages/studio/core/src/collab/
git commit -m "feat: wire CollabService into StudioService"
```

---

### Task 20: Add URL Route Handling for /r/{roomId}

**Files:**
- Modify: App routing configuration (read the routing setup first)

This task adds URL-based room joining. When a user navigates to `/r/{roomId}`, the app should show the JoinScreen and connect to that room.

**Step 1: Read the current routing setup**

Check `packages/app/studio/src/` for routing configuration.

**Step 2: Add `/r/:roomId` route that renders JoinScreen**

**Step 3: Verify build succeeds**

Run: `npx turbo build`
Expected: Build succeeds

**Step 4: Commit**

```bash
git add packages/app/studio/src/
git commit -m "feat: add /r/:roomId URL route for session joining"
```

---

## Phase 6: SpacetimeDB Client Integration

### Task 21: Install SpacetimeDB TypeScript SDK

**Step 1: Install the SDK**

Run: `cd packages/studio/core && npm install @clockworklabs/spacetimedb-sdk`

**Step 2: Verify build succeeds**

Run: `npx turbo build`
Expected: Build succeeds

**Step 3: Commit**

```bash
git add package-lock.json packages/studio/core/package.json
git commit -m "chore: install SpacetimeDB TypeScript SDK"
```

---

### Task 22: Create SpacetimeDB Connection Manager

**Files:**
- Create: `packages/studio/core/src/collab/stdb/StdbConnection.ts`
- Create: `packages/studio/core/src/collab/stdb/index.ts`
- Test: `packages/studio/core/src/collab/stdb/StdbConnection.test.ts`

This task creates the connection manager that wraps the SpacetimeDB client SDK. It handles connecting, reconnecting, and subscribing to table changes.

**Step 1: Write failing test for connection state management**

**Step 2: Implement StdbConnection with connect/disconnect/subscribe methods**

**Step 3: Run tests, verify pass**

**Step 4: Commit**

```bash
git add packages/studio/core/src/collab/stdb/
git commit -m "feat: add StdbConnection manager for SpacetimeDB client"
```

---

### Task 23: Wire SpacetimeDB to CollabService for Room Operations

**Files:**
- Modify: `packages/studio/core/src/collab/CollabService.ts`
- Modify: `packages/studio/core/src/collab/stdb/StdbConnection.ts`

This task connects the CollabService to SpacetimeDB so that `createRoom`, `joinRoom`, and `leaveRoom` actually call the SpacetimeDB reducers.

**Step 1: Add createRoom/joinRoom/leaveRoom methods to CollabService**

**Step 2: Wire room reducer calls through StdbConnection**

**Step 3: Subscribe to room_participant table changes for presence updates**

**Step 4: Subscribe to box_state table changes for BoxGraph sync**

**Step 5: Run tests, verify pass**

**Step 6: Commit**

```bash
git add packages/studio/core/src/collab/
git commit -m "feat: wire CollabService to SpacetimeDB room operations"
```

---

### Task 24: Wire WebRTC Signaling Through SpacetimeDB

**Files:**
- Modify: `packages/studio/core/src/collab/webrtc/PeerManager.ts`
- Create: `packages/studio/core/src/collab/webrtc/SignalingService.ts`

This task connects WebRTC signaling to the SpacetimeDB `webrtc_signal` table. When a peer joins, offer/answer/ICE candidates flow through SpacetimeDB.

**Step 1: Create SignalingService that subscribes to webrtc_signal table**

**Step 2: Implement offer/answer exchange flow**

**Step 3: Wire ICE candidate exchange**

**Step 4: Connect SignalingService to PeerManager**

**Step 5: Run tests, verify pass**

**Step 6: Commit**

```bash
git add packages/studio/core/src/collab/webrtc/
git commit -m "feat: add WebRTC signaling via SpacetimeDB tables"
```

---

### Task 25: Implement WebRTC Data Channel Asset Transfer Protocol

**Files:**
- Modify: `packages/studio/core/src/collab/assets/WebRTCAssetSource.ts`
- Modify: `packages/studio/core/src/collab/webrtc/PeerManager.ts`
- Test: Update `WebRTCAssetSource.test.ts`

This task implements the actual binary transfer protocol over WebRTC data channels.

**Step 1: Add data channel creation to PeerManager**

**Step 2: Implement request/response protocol in WebRTCAssetSource**

**Step 3: Add chunked transfer for large files (WebRTC data channels have ~256KB message limits)**

**Step 4: Add timeout handling for unresponsive peers**

**Step 5: Run tests, verify pass**

**Step 6: Commit**

```bash
git add packages/studio/core/src/collab/
git commit -m "feat: implement WebRTC data channel asset transfer protocol"
```

---

## Phase 7: End-to-End Integration

### Task 26: Wire Collaborate Menu to Real CollabService

**Files:**
- Modify: `packages/app/studio/src/service/StudioMenu.ts`

Replace the TODO comments in Task 16's menu items with actual CollabService calls.

**Step 1: Wire "Share Session..." to CollabService.createRoom() + ShareDialog**

**Step 2: Wire "Join Session..." to CollabService.joinRoom() + JoinScreen**

**Step 3: Wire "Leave Session" to CollabService.leaveRoom()**

**Step 4: Run dev server and manually test the flow**

Run: `npm run dev:studio`

**Step 5: Commit**

```bash
git add packages/app/studio/src/service/StudioMenu.ts
git commit -m "feat: wire Collaborate menu to CollabService"
```

---

### Task 27: Integrate CursorOverlay and ParticipantPanel into Main UI

**Files:**
- Modify: Main app layout component (read structure first)

**Step 1: Read the main layout component to find where to inject the overlay**

**Step 2: Add CursorOverlay as a child of the main workspace area**

**Step 3: Add ParticipantPanel to the header/toolbar area**

**Step 4: Subscribe to PresenceService.onChange to re-render cursors**

**Step 5: Add mouse/pointer event listener to send local cursor position (throttled to 10Hz)**

**Step 6: Commit**

```bash
git add packages/app/studio/src/
git commit -m "feat: integrate cursor overlay and participant panel into main UI"
```

---

### Task 28: Add S3 Source to Asset Transport Chain

**Files:**
- Modify: `packages/studio/core/src/collab/CollabService.ts`

**Step 1: Read S3 config from localStorage**

**Step 2: If S3 is configured, insert S3AssetSource between OpfsAssetSource and WebRTCAssetSource in the chain**

**Step 3: When creator publishes an asset, upload to S3 if configured and call update_asset_s3_url reducer**

**Step 4: Run tests, verify pass**

**Step 5: Commit**

```bash
git add packages/studio/core/src/collab/
git commit -m "feat: add optional S3 source to asset transport chain"
```

---

### Task 29: Deploy SpacetimeDB Module & End-to-End Test

**Step 1: Publish the SpacetimeDB module**

Run: `cd packages/server/spacetimedb-module && spacetime publish opendaw --server <your-server>`

**Step 2: Update default endpoint in CollabSettings.tsx**

**Step 3: Run dev server with two browser windows**

Run: `npm run dev:studio`

**Step 4: Test full flow:**
- Window 1: Click "Share Session..." → copy URL
- Window 2: Open URL → enter name → join
- Verify BoxGraph changes sync both ways
- Verify cursors appear in both windows
- Verify sample assets transfer via WebRTC

**Step 5: Commit any fixes**

```bash
git commit -m "fix: end-to-end collaboration integration fixes"
```

---

## Summary

| Phase | Tasks | Focus |
|-------|-------|-------|
| 1: Server Module | 1-3 | SpacetimeDB Rust tables & reducers |
| 2: Client Services | 4-10 | RoomService, AssetTransport, PresenceService, CollabService |
| 3: UI Components | 11-15 | ShareDialog, JoinScreen, ParticipantPanel, CursorOverlay |
| 4: Menu & Routing | 16-18 | StudioMenu, S3Settings, CollabSettings |
| 5: Integration | 19-20 | Wire services into app, URL routing |
| 6: SpacetimeDB Client | 21-25 | SDK install, connection manager, signaling, data transfer |
| 7: End-to-End | 26-29 | Full wiring, deployment, testing |

Total: **29 tasks**, TDD throughout, frequent commits.
