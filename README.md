# sekiro-coop

Two-player seamless co-op mod for *Sekiro: Shadows Die Twice* (v1.06), written in Rust and injected as a DLL via [me3](https://github.com/garyttierney/me3). Sekiro has no native multiplayer, so everything, networking, state sync, authority, event replication, is built from scratch on top of function hooks into the game process.

## Status

The **data plane is validated** against a live Sekiro instance plus a `peer-simulator` standing in as the second player: DLL injection, six native hooks, the UDP session with handshake and reliability, bidirectional player-snapshot sync at 60 Hz, edge-triggered game-event replication, a 42-entity enemy registry with host-authoritative state broadcast, and HP write-back through a validated pointer chain all work end-to-end without crashing on the happy path.

The mod is **not yet end-user playable**. The open blocker is rendering the remote player in the local world: writing positions into `ChrIns` memory is accepted but ignored by the render path, and the native functions behind the EMEVD opcodes that could drive a puppet NPC (animation playback, character warp) have no known AOBs. The planned unblock is patching `common.emevd` so the game's own event VM dispatches those natives, see [docs/HANDOFF.md](docs/HANDOFF.md) for the full plan and current state.

## Architecture

The workspace is layered, lowest to highest:

| Crate | Layer |
|---|---|
| `sekiro-sdk-sys` | Memory substrate: AOB patterns, per-version offsets (`V1_02`, `V1_06`), pointer chains, native-function resolution, live `ChrIns` readers/writers, param-repository walker, Cheat Engine table XML ingestion |
| `sekiro-sdk-core` | Typed layer: entity wrappers, enums (`TeamType`, `MultiplayerState`, ...), character/item catalogs, `AtkParam`/`SpEffect` types, and the MinHook FFI shim (`hook.rs`, behind the `minhook` feature) |
| `sekiro-sdk-bridge` | `BridgeEvent` dispatcher: translates hook callbacks (flags, effects, items, XP, warps) into a typed event stream and back |
| `sekiro-coop-rollback` | Snapshot ring, state-delta compression with spawn/despawn tracking, predictor, re-simulation, and the `ChrInsStepper` write-back path. Unit-tested; **not yet wired into live play** |
| `sekiro-coop-authority` | Authority table (host owns enemies), proximity-based handoff with hysteresis, deterministic PCG RNG keyed on `(match_seed, frame, call_site)` |
| `sekiro-coop-net` | UDP transport, session/handshake, wire format, reliability layer, desync detector, Steam P2P stub (behind the `steam` feature) |
| `sekiro-coop-dll` | The injected `cdylib`: DLL entry, detour installation, 60 Hz fallback ticker, optional hudhook ImGui overlay (behind the `overlay` feature) |
| `sekiro-coop-emevd` | Structured EMEVD binary reader/writer with a 386-instruction catalog, SOLO-branch promotion, and custom-event injection; library plus CLI |

### Hooking

Function hooking uses [MinHook](https://github.com/TsudaKageyu/minhook), vendored in-tree (sources at `vendor/minhook/`, prebuilt static lib at `vendor/lib/MinHook.x64.lib`). Native function addresses are resolved by AOB scan at startup. Six detours are installed and validated live: `SetFlag`, `ApplyEffect`, `DeleteEffect`, `GiveItem`, `AddExperience`, `WarpBonfire`.

Two design points worth knowing:

- The enemy registry is fed from hook arguments (e.g. the entity pointer `ApplyEffect` receives) instead of scanning `WorldChrMan`, pointer sweeps repeatedly crashed on stale data, while hook-supplied pointers are known-good by construction.
- Remote events are applied through trampoline-direct calls to the original functions, so applying a replicated event never re-enters the detour and echoes back over the wire.

### Networking / data plane

Raw UDP, peer-to-peer, configured by environment variables (no matchmaking UI yet):

- **Session**: magic + versioned header, handshake, bincode-serialized `PacketBody`. Note: bincode enum tags mean any `wire.rs` change requires rebuilding both peers.
- **Reliability**: sliding-window ACKs with sequence numbers, ack bitmap, and a retransmit queue, for packet types that need it.
- **PlayerSnapshot**: full own-player state, 60 Hz, unreliable, bidirectional.
- **BridgeEvents**: flag/effect/item/XP/warp events, edge-triggered (sent only on change), reliable. Application on the receiving side is gated behind `SEKIRO_COOP_APPLY_REMOTE=1`.
- **EnemyStates**: host-only broadcast at 5 Hz, delta-filtered via a 64-bit per-entity digest (~97% of entries skipped on idle frames). The client resolves remote handles to local `ChrIns` pointers and applies HP decrement-only through the validated state-module chain.
- **Authority**: host owns enemies, broadcasts their state and ignores inbound enemy claims; the client mirrors and only broadcasts its own player.
- **Desync detection**: 60-frame state-hash exchange with a three-strike session kill.

## Building

Prerequisites:

- Rust 1.78 (pinned via `rust-toolchain.toml`); the DLL needs Windows with the MSVC toolchain (`x86_64-pc-windows-msvc`)
- Nothing else, MinHook is vendored, and the build script picks up `vendor/lib/MinHook.x64.lib` automatically (set `MINHOOK_LIB_DIR` to override with your own build)

```powershell
# The mod DLL -> target\release\sekiro_coop.dll
cargo build --release -p sekiro-coop-dll

# With the ImGui overlay (pulls in hudhook's DX11 deps)
cargo build --release -p sekiro-coop-dll --features overlay

# Offline tools
cargo build --release -p sekiro-coop-emevd -p aob-scanner -p determinism-probe -p live-inspector -p peer-simulator
```

Tools:

- `peer-simulator`, stands in for a second Sekiro instance over UDP; completes the handshake and streams snapshots (see `run-peer.cmd`)
- `aob-scanner`, runs every documented AOB pattern against a `sekiro.exe` and reports hits
- `live-inspector`, offline pointer-chain walker
- `determinism-probe`, diffs two snapshot dumps
- `sekiro-coop-emevd`, the EMEVD patcher CLI

## Testing

```powershell
cargo test --workspace --exclude sekiro-coop-dll
```

100 unit and property tests covering the wire protocol, reliability, delta compression, desync detection, EMEVD format round-trips, CE-table parsing, the rollback stepper, proximity handoff, and RNG determinism. Everything except the DLL crate is platform-independent, the suite also builds and passes on Linux.

For live wire-path testing without two game installs, run the DLL in one Sekiro instance and `peer-simulator` as the other end (`--fake-events` sends synthetic BridgeEvents; the echo path exercises the enemy-state apply logic).

## Installing

Requires a legal copy of Sekiro v1.06 on Steam and [me3](https://github.com/garyttierney/me3) (a known-good me3 v0.11.0 is checked in under `tools/me3/`). The launcher runs the game with Arxan disabled and online blocked, keep it that way.

1. Build the DLL (above) and stage it at `tools/me3/sekiro-coop-mods/sekiro_coop.dll`.
2. Use the checked-in profile `tools/me3/sekiro-coop.me3` (loads the DLL as a native and `sekiro-coop-mods/` as an asset-override package), or adapt `dist/me3-profile.toml`.
3. Configure the connection via environment variables and launch through me3 on both machines.

`launch.ps1` automates all of this on Windows: preflight checks, build, stage, env setup, and launch (`.\launch.ps1 -Peer host` / `-Peer client`). `kill-sekiro.ps1` is the clean-shutdown helper.

| Variable | Purpose |
|---|---|
| `SEKIRO_COOP_PEER` | `host` or `client`; drives authority behaviour |
| `SEKIRO_COOP_BIND` | UDP bind address, e.g. `0.0.0.0:28000` |
| `SEKIRO_COOP_PEER_ADDR` | Peer endpoint, e.g. `203.0.113.42:28000` |
| `SEKIRO_COOP_LOG` | Tracing filter; default `info,sekiro_coop=debug` |
| `SEKIRO_COOP_APPLY_REMOTE` | `1` to enable applying remote events/HP writes |

Logs land in `%LOCALAPPDATA%\sekiro-coop\sekiro-coop.log`.

For EMEVD patching (`common.emevd` two-player promotion and custom events), decompress the `.dcx` with Yabber, run the `sekiro-coop-emevd` CLI, re-pack, and serve the result through me3's file override, step-by-step in [docs/INSTALL.md](docs/INSTALL.md) and [docs/HANDOFF.md](docs/HANDOFF.md).

## Repository layout

```
sekiro-coop/
├── crates/
│   ├── sekiro-sdk-sys/        # AOBs, offsets, memory, natives, live ChrIns I/O, CE-table XML
│   ├── sekiro-sdk-core/       # Typed entities, enums, catalogs, MinHook shim
│   ├── sekiro-sdk-bridge/     # BridgeEvent dispatcher (combat / AI / world)
│   ├── sekiro-coop-rollback/  # Snapshots, deltas, predictor, re-sim, stepper
│   ├── sekiro-coop-authority/ # Authority table, handoff, seeded RNG
│   ├── sekiro-coop-net/       # Transport, session, wire, reliability, desync
│   ├── sekiro-coop-dll/       # DLL entry, detours, ticker, overlay
│   └── sekiro-coop-emevd/     # EMEVD reader/writer + patcher CLI
├── tools/
│   ├── peer-simulator/        # Second-Sekiro stand-in over UDP
│   ├── aob-scanner/           # Validates AOB patterns against sekiro.exe
│   ├── live-inspector/        # Offline pointer-chain walker
│   ├── determinism-probe/     # Twin-instance snapshot diff
│   └── me3/                   # Vendored me3 v0.11.0 + profile + mods staging dir
├── vendor/
│   ├── minhook/               # MinHook sources (BSD-2-Clause)
│   └── lib/MinHook.x64.lib    # Prebuilt static lib the build script links
├── dist/me3-profile.toml      # Template me3 profile
├── docs/
│   ├── HANDOFF.md             # Current state, decision log, next steps (most up to date)
│   ├── GAPS.md                # Original gap inventory (partially superseded by HANDOFF)
│   └── INSTALL.md             # Build / install / troubleshooting detail
├── launch.ps1                 # Preflight + build + stage + launch via me3
└── run-peer.cmd               # peer-simulator launcher
```

## Known limitations

- **No visible remote player yet.** The puppet-rendering blocker described above; the EMEVD-patch pipeline (UXM + Yabber) is the planned fix and has not been completed.
- **No damage-application hook.** Its AOB is unknown, so client-side hits on bosses rely on the host's authoritative state broadcast.
- **Raw UDP only.** Direct IP between two peers; Steam P2P is an empty feature stub. No NAT traversal, no matchmaking UI.
- **Rollback is not live.** The snapshot/delta/re-sim machinery is implemented and tested but the live loop currently runs snapshot-sync, not rollback.
- **Sekiro v1.06 only** has validated offsets; other versions are scaffolded in `offsets.rs` but unverified.
- **Two players**, by design of the authority model.

## License

AGPL-3.0-or-later. Vendored components keep their own licenses: MinHook (BSD-2-Clause, `vendor/minhook/LICENSE.txt`), me3 binaries (MIT/Apache-2.0, `tools/me3/`).
