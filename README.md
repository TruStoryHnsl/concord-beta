# Concord (beta)

> Archived prototype. Not maintained. Superseded by [concord](https://github.com/TruStoryHnsl/concord).

This is the abandoned v2 prototype of [Concord](https://github.com/TruStoryHnsl/concord), exploring a peer-to-peer mesh transport built on **Rust + Tauri 2 + libp2p** instead of the centralized Matrix homeserver model used by current Concord.

The experiment was discontinued in favor of the Matrix-based approach. The code is preserved here as a reference for the design decisions, the libp2p mesh topology work, and the cross-platform Tauri shell.

## What it tried to be

A self-organizing voice/text chat mesh where every client was also a peer — no central server, no homeserver, no SFU. Nodes discovered each other over libp2p, exchanged Ed25519 identity, and routed messages directly through the mesh.

## Why it was archived

Mesh transport made several common chat features (discovery at scale, federation, persistent history, moderation, identity recovery) significantly harder than the equivalents on Matrix, and the Matrix-based architecture in current [concord](https://github.com/TruStoryHnsl/concord) landed first. See [`ARCHITECTURE.md`](./ARCHITECTURE.md) for the full design and the trade-offs that led to discontinuation.

## Tech stack

| Layer | Choice |
|-------|--------|
| Shell | Tauri 2 |
| Backend | Rust 2024 workspace (8 crates) |
| Transport | libp2p mesh |
| Identity | Ed25519 |
| Frontend | Web UI bundled into the Tauri shell |

## Workspace layout

```
concord-beta/
├── crates/           # 8 Rust crates (core, net, media, store, webhost, daemon, poc, ...)
├── src-tauri/        # Tauri 2 desktop shell
├── frontend/         # Web UI bundled into the Tauri shell
├── scripts/          # Build and dev scripts (Linux, desktop, mobile, server)
├── design/           # Mockups and the kinetic-node design language
├── .github/          # CI workflows
├── ARCHITECTURE.md   # Full architecture document
├── Cargo.toml        # Workspace root
└── LICENSE           # MIT
```

## Build (for the curious)

```bash
cargo build --workspace
```

This repository is preserved as a reference, **not** a maintained project. Issues and PRs will not be triaged.

## License

MIT — see [LICENSE](./LICENSE).
