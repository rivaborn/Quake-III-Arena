# code/cgame/cg_public.h

## File Purpose
Defines the public interface contract between the cgame module (client-side game logic) and the main engine executable. It declares the snapshot data structure and enumerates all syscall IDs for both engine-to-cgame (imported) and cgame-to-engine (exported) function dispatch tables.

## Core Responsibilities
- Define `snapshot_t`, the primary unit of server-state delivery to the client
- Enumerate all engine services available to the cgame VM via `cgameImport_t` trap IDs
- Enumerate all cgame entry points callable by the engine via `cgameExport_t`
- Define `CMD_BACKUP` / `CMD_MASK` constants for the client command ring buffer
- Declare `CGAME_IMPORT_API_VERSION` for ABI compatibility checking
- Declare cgame UI event type constants (`CGAME_EVENT_*`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `snapshot_t` | struct | One server-authoritative world snapshot: player state, visible entities, area mask, timing metadata |
| `cgameImport_t` | enum | Syscall dispatch IDs for engine functions callable from cgame (renderer, sound, CM, FS, key, cinematic, etc.) |
| `cgameExport_t` | enum | Entry point IDs for cgame functions invoked by the engine (init, draw, input events, shutdown) |

## Global / File-Static State
None.

## Key Functions / Methods
This is a header-only interface file; no function implementations are present. All callable interfaces are represented as integer enum values used in VM syscall dispatch.

### snapshot_t fields (notable members)
- `snapFlags` — bitmask (e.g. `SNAPFLAG_RATE_DELAYED`) describing delivery conditions
- `ping` — round-trip latency at snapshot time
- `serverTime` — authoritative server timestamp in milliseconds
- `areamask[MAX_MAP_AREA_BYTES]` — PVS/portal-area visibility bitmask
- `ps` (`playerState_t`) — full local player state
- `entities[MAX_ENTITIES_IN_SNAPSHOT]` (`entityState_t`) — up to 256 visible entities
- `numServerCommands` / `serverCommandSequence` — pending reliable server-to-client text commands

## Control Flow Notes
- **Init**: The engine calls `CG_INIT` (via `cgameExport_t`) after map load or renderer restart, passing `serverMessageNum`, `serverCommandSequence`, and `clientNum`.
- **Per-frame**: The engine calls `CG_DRAW_ACTIVE_FRAME` each rendered frame; cgame pulls the current snapshot via `CG_GETSNAPSHOT` and drives the entire scene submission pipeline.
- **Shutdown**: `CG_SHUTDOWN` is called to allow cgame to flush open files.
- **Input**: `CG_KEY_EVENT` and `CG_MOUSE_EVENT` route raw input; `CG_EVENT_HANDLING` manages UI overlay modes (scoreboard, team menu, HUD editor).
- **Syscall model**: cgame runs as a QVM; all engine access goes through integer trap IDs (`cgameImport_t`), dispatched by the engine's `CG_SystemCalls` handler defined elsewhere.

## External Dependencies
- `MAX_MAP_AREA_BYTES` — defined in `qcommon/qfiles.h` or `game/q_shared.h`
- `playerState_t` — defined in `game/bg_public.h`
- `entityState_t` — defined in `game/q_shared.h` / `game/bg_public.h`
- `byte`, `qboolean`, `stereoFrame_t` — defined in `game/q_shared.h`
- `SNAPFLAG_*` constants — defined elsewhere (likely `qcommon/qcommon.h`)
- All `cgameImport_t` trap implementations — defined in `client/cl_cgame.c` (`CL_CgameSystemCalls`)
- All `cgameExport_t` entry points — implemented in `cgame/cg_main.c` (`vmMain`)
