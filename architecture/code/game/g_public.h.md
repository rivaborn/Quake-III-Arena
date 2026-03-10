# code/game/g_public.h

## File Purpose
Defines the public interface contract between the Quake III game module (QVM) and the server engine. It declares server-visible entity flags, shared entity data structures, and the complete syscall tables for both engine-to-game (imports) and game-to-engine (exports) communication.

## Core Responsibilities
- Define `GAME_API_VERSION` for versioning the game/server ABI
- Declare `SVF_*` bitflags controlling server-side entity visibility and behavior
- Define `entityShared_t` and `sharedEntity_t` as the shared memory layout the server reads directly
- Enumerate all engine syscalls available to the game module (`gameImport_t`)
- Enumerate all entry points the server calls into the game module (`gameExport_t`)
- Expose BotLib syscall ranges (200–599) as part of the game import table

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `entityShared_t` | struct | Server-readable portion of a game entity: link state, flags, bounds, origin, owner |
| `sharedEntity_t` | struct | Top-level layout combining `entityState_t s` and `entityShared_t r`; server accesses `gentity_t` through this pointer |
| `gameImport_t` | enum | Syscall indices for all engine services callable by the game VM (general + server + BotLib) |
| `gameExport_t` | enum | Entry-point indices the server invokes on the game module per frame/event |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining constants, types, and enumerations.

## Control Flow Notes
- At startup the server calls `GAME_INIT` (export 0); each frame it calls `GAME_RUN_FRAME` and `GAME_CLIENT_THINK` per client.
- The server locates all `gentity_t` instances via `G_LOCATE_GAME_DATA` (import), then reads them as `sharedEntity_t *` directly — no function call overhead for per-entity data.
- BotLib imports occupy three reserved ranges: general botlib (200–299), AAS (300–399), EA/AI (400–599), keeping them from colliding with engine syscalls.
- `G_LINKENTITY` / `G_UNLINKENTITY` must bracket any entity that participates in collision or snapshot visibility; entities not linked are invisible to both the network layer and collision system.

## External Dependencies
- `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `vec3_t`, `vmCvar_t`, `qboolean` — defined in `q_shared.h` / `bg_public.h` (game-shared layer)
- `gentity_t` — defined in `g_local.h`; `g_public.h` only sees it as a forward-referenced pointer target through `sharedEntity_t`
- Server engine — consumes this header to understand entity layout and dispatch the VM syscall tables
- BotLib — its full API surface is tunneled through the `gameImport_t` enum rather than direct linking
