# Concord (beta)

> Active R&D — native peer-to-peer mesh chat platform. A parallel research track to [concord](https://github.com/TruStoryHnsl/concord) (production, Matrix-based).

Concord (beta) is an experimental implementation of **native P2P mesh chat** built on Rust + Tauri 2 + libp2p. Where the production [concord](https://github.com/TruStoryHnsl/concord) uses a central Matrix homeserver, concord-beta is exploring whether the same product — text, voice, presence, trust, the works — can be delivered with **no central server at all**. Every device runs the same code and participates as both client and host.

This repository documents an architecture in active development. The design goals below describe the target; the [current roadblocks](#current-roadblocks) describe the open problems being worked on right now.

**Status as of 2026-04-05:** ~27,800 LOC (16,893 Rust + 10,921 TypeScript), 275 Rust tests passing, validated cross-network PoC (LAN ↔ WireGuard tunnel) between two physical machines.

---

## Architectural design goals

### 1. No central server, ever

No homeserver, no SFU, no message broker, no required relay. Every peer is fully capable of being a host. The only optional infrastructure is a WireGuard mesh VPN ([orrtellite](https://github.com/TruStoryHnsl/orrtellite) / Tailscale / Headscale) for traversing NATs across the open internet — and even that should become unnecessary once the infrastructure-free transports land.

### 2. Single identity, single key

One Ed25519 keypair per node serves as both the application identity *and* the libp2p network identity. The same key signs messages, generates the libp2p `PeerId`, signs trust attestations, and encrypts the at-rest identity store. There are no separate user accounts, server logins, or homeserver registrations.

```
Ed25519 Secret Key (32 bytes)
    ├─► Concord Peer ID:  hex(public_key)            "729af2a2..."
    ├─► libp2p PeerId:    multihash(public_key)      "12D3KooWHXjg..."
    ├─► Signing:          messages, attestations
    └─► Storage:          ChaCha20-Poly1305 encrypted with device key
```

### 3. Five-tier transport hierarchy

A node should automatically pick the best available transport at any moment, including infrastructure-free options for off-grid or mesh-only use:

| Tier | Technology | Bandwidth | Infrastructure | Status |
|------|-----------|-----------|----------------|--------|
| **BLE** | Bluetooth LE | ~200 kbps | None | Trait defined, plugin needed |
| **WiFi Direct** | WiFi P2P | ~250 Mbps | None | Trait defined, plugin needed |
| **WiFi AP** | Device hotspot | ~100 Mbps | None | Trait defined, plugin needed |
| **LAN** | mDNS over IP + QUIC | Full | Router | **Working** |
| **Tunnel** | QUIC over WireGuard mesh | Full | Internet | **Working** |

### 4. Three communication pathways

| Pathway | Scope | GossipSub Topic | Encryption |
|---------|-------|-----------------|------------|
| **Forums** | Mesh-wide, public | `concord/forum/{local,global}` | Well-known seed (intentional — "encrypted radio") |
| **Servers / Places** | Org-scoped | `concord/{server_id}/{channel_id}` | Per-server key, X25519 + ChaCha20-Poly1305 |
| **Direct (1:1)** | Two peers | `concord/dm/{sorted_peer_pair}` | X25519 DH + ChaCha20-Poly1305 |

Local forums propagate via TTL-bounded flood (`hop_count` / `max_hops`); global forums ride native GossipSub gossip with no hop limit.

### 5. Defense in depth — six independent encryption layers

| Layer | Scope | Algorithm | Key Management |
|-------|-------|-----------|----------------|
| libp2p Transport | Every connection | QUIC + Noise | Ephemeral per-connection |
| WireGuard | Tunnel connections | WireGuard (OS) | Headscale-managed |
| Channel Messages | Server channels | ChaCha20-Poly1305 | HMAC-derived from server secret |
| Forum Messages | Forum scope | ChaCha20-Poly1305 | Well-known seed (deliberate) |
| Direct Messages | 1:1 conversations | X25519 + ChaCha20-Poly1305 | DH key exchange |
| Identity Storage | At rest | ChaCha20-Poly1305 | Device key file |

Compromising any one layer leaves the others intact.

### 6. Mesh map as a distributed database

Every entity (node, place, call, locale) has a deterministic 32-byte address derived via `HMAC-SHA256("concord-mesh-address-v1", identifier)`. Map entries gossip across the mesh with **confidence tiers** that decay without re-verification:

| Tier | Weight | Source |
|------|--------|--------|
| SelfVerified | 1.0 | Data about yourself |
| TunnelVerified | 0.75 | Exchanged over a trusted tunnel |
| ClusterVerified | 0.5 | Verified by cluster consensus |
| Speculative | 0.25 | Received from an untrusted source |

Sync is a three-phase gossip protocol over `concord/mesh/map-sync`: digest broadcast every 60 s → delta request when remote has newer data → delta response (max 50 entries per batch). Friends get a 15 s sync cooldown and automatic confidence upgrades.

### 7. Web of trust

Five tiers (Unverified → Recognized → Established → Trusted → Backbone) backed by signed Ed25519 attestations broadcast over `concord/mesh/attestations`. Attestation weight scales with the attester's own trust level (0.5×–3.0×), so a backbone node's vouch is worth six times an unknown peer's. Trust score is a weighted positive/negative ratio clamped to `[-1.0, 1.0]`. Cross-account reputation bleed (15% sibling factor) ties multiple aliases of the same identity together.

---

## Workspace layout

```
concord-beta/
├── crates/
│   ├── concord-core/       # Identity, types, crypto, wire format, trust, mesh map
│   ├── concord-net/        # libp2p swarm, discovery, GossipSub, tunnel detection
│   ├── concord-media/      # Voice/video capture, Opus codec, signaling
│   ├── concord-store/      # SQLite persistence (24 tables, WAL mode)
│   ├── concord-webhost/    # Browser guest access via axum + WebSocket
│   ├── concord-daemon/     # Headless server binary
│   └── concord-poc/        # Cross-machine LAN + tunnel test binary
├── src-tauri/              # Tauri 2 desktop / mobile shell
├── frontend/               # React 19 + TypeScript + Zustand + Tailwind
├── scripts/                # Build scripts (Linux, desktop, mobile, server)
├── design/                 # Mockups + Kinetic Node design language
├── .github/                # CI workflows
├── ARCHITECTURE.md         # Full architecture document (370 lines)
├── Cargo.toml              # Workspace root
└── LICENSE                 # MIT
```

| Crate | Key Dependencies |
|-------|-----------------|
| `concord-core` | `ed25519-dalek`, `x25519-dalek`, `chacha20poly1305`, `rmp-serde`, `hmac`, `sha2` |
| `concord-net` | `libp2p 0.54` (mDNS, Kademlia, GossipSub, QUIC, Noise, Yamux, Relay, DCUtR, Identify) |
| `concord-store` | `rusqlite 0.32` (bundled SQLite, WAL mode) |
| `concord-media` | `str0m 0.7` (WebRTC), `cpal 0.17` (audio I/O), `audiopus 0.3` (Opus codec) |
| `concord-webhost` | `axum 0.8`, `rust-embed 8` |
| `src-tauri` | `tauri 2` |

---

## What's working today

Validated end-to-end:

- **Identity lifecycle** — Ed25519 keypair generation, encrypted persistence, libp2p PeerId unification
- **Alias system** — multiple personae per identity, GossipSub broadcast, peer caching
- **LAN mesh networking** — mDNS auto-discovery (<1 s), QUIC connections
- **WireGuard tunnel networking** — `tailscale status --json` parsing, cross-network dial, message delivery
- **GossipSub messaging** — subscribe, publish, receive, decrypt, content-addressed dedup
- **Channel encryption** — server keys, X25519 key exchange, ChaCha20-Poly1305
- **Server management** — create, join, leave, channels, members, invites with `max_uses`
- **Direct messages** — X25519 DH, end-to-end encrypted, persisted history
- **Forums** — local hop-limited (TTL flood) + global (native GossipSub gossip), encrypted
- **Friends** — request/accept, presence heartbeat, encrypted signals
- **Trust system** — attestation signing/verification, weighted scoring, 5-tier badges
- **Voice audio** — `cpal` capture → Opus encode → GossipSub publish → Opus decode → `cpal` playback (works for 1:1)
- **SQLite persistence** — 24 tables, schema migrations, WAL mode, foreign keys
- **Frontend** — 50+ React components, 11 Zustand stores, Tauri IPC + browser mock layer
- **Headless daemon** — CLI, TOML config, admin API
- **Multi-platform scaffolds** — iOS (Xcode), Android (Gradle), macOS, Linux, Windows
- **275 Rust tests** across all crates, all passing

### PoC validation results

**Test 1 — LAN discovery (orrion ↔ orrgate, same subnet):**
mDNS auto-discovery in <1 s, QUIC connection on `192.168.1.x`, GossipSub message delivered, classified as `local_mdns`.

**Test 2 — Tunnel communication (orrion on home WiFi ↔ cb17 on cellular hotspot):**
LAN unreachable (`192.168.1.166` — 100% packet loss). Tunnel reachable (`100.116.151.17` — ~80 ms). QUIC connection established over WireGuard mesh IPs, GossipSub message delivered, classified as `wireguard`.

---

## Current roadblocks

These are the open problems currently being worked on, listed in roughly descending order of how blocking each is for "feature parity with a centralized chat system." None of these are reasons the project is stuck — they are the work in progress.

### 1. No message history sync (CRITICAL)

`concord-net/src/sync.rs` is currently a stub. Messages broadcast via GossipSub are delivered exactly once — if a peer is offline (phone asleep, laptop closed, network down), they miss the message permanently with no recovery mechanism.

For a mesh network where most devices are intermittently online, this is the single biggest gap.

**Needed:** a catch-up protocol that lets a peer ask its neighbors *"what did I miss in topic X since timestamp Y?"* and receive a verified delta. Should reuse the mesh-map sync protocol's digest/delta pattern but operate on message streams instead of map entries.

### 2. No CRDT / distributed server state (HIGH)

The architectural vision is *"the server exists on the mesh and all connected nodes share it"* — but the implementation stores server state (members, channels, bans) in each node's local SQLite with no cross-node replication. Two nodes that both add a member while disconnected will end up with divergent member lists and no reconciliation path.

**Needed:** integrate a CRDT library (Automerge, Yjs port, or a custom op-based CRDT over GossipSub) and rebuild the server-state model around it. Vector clocks, causal ordering, and conflict resolution are all currently unimplemented.

### 3. Voice pipeline doesn't scale past two participants (HIGH for voice)

Voice currently flows as Opus frames over GossipSub at 100 fps per peer. The pipeline works end-to-end (cpal → Opus → GossipSub → Opus → cpal), but it's an O(n²) bandwidth pattern that breaks at ~3+ participants.

`concord-media/src/sfu.rs` is comment-only. SDP signaling uses placeholder strings — there are no actual `str0m` WebRTC sessions.

**Needed:** implement a real SFU on top of `str0m` with selective forwarding, plus real ICE/SDP negotiation. The voice state machine (join/leave/mute/deafen, participant tracking) is in place; the media plane is not.

### 4. Server keys not distributed to joining members (HIGH)

When a server is created, the 32-byte secret is stored locally on the creator's node. When another peer joins via invite, the server key is **not** transmitted to them. Joining members can't decrypt channel messages because they lack the key.

The `KeyRequest` / `KeyResponse` X25519 handshake message types exist but are not wired into the join flow.

**Effect today:** only the server creator can read messages in their own server.

### 5. Mesh map sync not wired to persistence (HIGH)

The three-phase digest/delta sync protocol works in memory, but the in-memory map is never written to SQLite. On node restart the entire mesh map is lost and has to be re-gossiped from neighbors. The `mesh_map_entries` and `mesh_map_tombstones` tables exist in the schema but are unused by the sync code.

**Needed:** wire `MeshMap::apply_delta()` and tombstone handling through the store layer with proper transaction boundaries.

### 6. Trust attestation signatures not verified on receipt (MEDIUM, security)

Attestations are Ed25519-signed by the attester and broadcast over GossipSub. Receiving nodes store them but **do not verify the signature**, because the attester's public key may not be locally known yet at receipt time. A malicious node could currently forge attestations against any subject.

**Needed:** verify signatures lazily when the attester's pubkey becomes known (via the Identify protocol or a Kademlia lookup), and discard attestations whose signatures fail. Until then, the trust score should treat unverified attestations with reduced weight.

### 7. No perfect forward secrecy in DMs (MEDIUM)

DM sessions use static X25519 shared secrets. Compromise of a long-lived key reveals every past and future DM in that session.

**Needed:** Double Ratchet (Signal protocol) for per-message key rotation. The `crypto.rs` module is already structured to accommodate ratcheting; only the state machine and key-rotation policy are missing.

### 8. Infrastructure-free transports unimplemented (MEDIUM, off-grid use case)

The `Transport` trait is defined and `TransportManager` knows about all five tiers, but BLE, WiFi Direct, and WiFi AP are not implemented. Each requires a platform-native Tauri 2 plugin:

| Platform | BLE | WiFi Direct / P2P |
|----------|-----|-------------------|
| iOS / macOS | CoreBluetooth | MultipeerConnectivity |
| Linux | BlueZ | wpa_supplicant |
| Android | Android BLE API | Nearby Connections |
| Windows | WinRT Bluetooth | WinRT WiFi Direct |

Until these land, concord-beta requires either a LAN or a WireGuard tunnel to operate.

### 9. No file / media sharing (MEDIUM, feature gap)

Messages are text-only. There is no file upload, no image attachment, no thumbnail generation, no chunked transfer.

**Needed:** content-addressed storage (probably IPFS-style CIDs) plus a chunked transfer protocol — either a stream-based libp2p protocol or a chunked GossipSub flow with reassembly.

### 10. No content moderation tools (LOW–MEDIUM)

Forums have no pinning, no thread lock, no removal, no per-user mute. The design is intentionally censorship-resistant for *public mesh forums*, but server admins of *named places* should still have moderation tools for their own spaces. The data model supports per-message authorship and per-server roles; what's missing is the policy layer and the UI.

### 11. No notifications

No push notification infrastructure (APNs, FCM), no local OS notifications, no badge counts. The frontend has UI states for unread/mention but nothing drives them.

### 12. Forum encryption is intentionally weak

`derive_forum_key()` uses a hardcoded seed (`"concord-forum-well-known-seed-v1"`) so every Concord node can decrypt every forum message. This is the *"encrypted radio"* model — external observers can't read GossipSub traffic, but any peer running Concord can. This is a deliberate design choice for public mesh forums, but it means there is no per-forum access control. Whether this becomes a roadblock depends on whether private forums get prioritized as a future feature.

---

## Build

```bash
cargo build --workspace               # all crates
cargo run -p concord-poc               # PoC binary (cross-machine LAN + tunnel test)
cargo tauri dev                        # native app with hot-reloading frontend
cd frontend && npm run dev             # browser-only UI iteration with mock data
cargo test --workspace                 # 275 tests
```

### Mobile

iOS and Android scaffolds are generated under `src-tauri/gen/`. Building for those targets requires the matching toolchains (Xcode for iOS, Android NDK + Gradle for Android).

```bash
cargo tauri ios dev
cargo tauri android dev
```

See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the full architectural reference: crate graph, swarm composition, identity model, mesh map sync protocol, encryption layers, voice pipeline, and PoC validation results.

---

## Relationship to current concord

[concord](https://github.com/TruStoryHnsl/concord) is the production-ready chat platform built on Matrix (Tuwunel homeserver, FastAPI backend, LiveKit voice, React web client + Tauri desktop app). It is the recommended way to actually run a Concord instance today.

Concord (beta) is an **alternative architecture** being researched in parallel — same product vision, fundamentally different transport. The two repositories share the *Concord* name and product goals but are independent codebases with no shared code at this time. If concord-beta closes the roadblocks above, the architectures may converge or merge. Until then, treat current concord as production and concord-beta as research.

---

## License

MIT — see [LICENSE](./LICENSE).
