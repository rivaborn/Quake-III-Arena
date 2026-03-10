# code/game/botlib.h

## File Purpose
Defines the public API boundary between the Quake III game module and the bot AI library (botlib). It declares all function pointer tables (vtables) used to import engine services into botlib and export bot subsystem capabilities back to the game.

## Core Responsibilities
- Define the versioned `botlib_export_t` / `botlib_import_t` interface structs
- Declare input/state types (`bot_input_t`, `bot_entitystate_t`, `bsp_trace_t`) shared across the boundary
- Group bot subsystem exports into nested vtable structs: `aas_export_t`, `ea_export_t`, `ai_export_t`
- Define action flag bitmasks used to encode bot commands
- Define error codes (`BLERR_*`) and print type constants for botlib diagnostics
- Document all configurable library variables and their defaults in a reference comment block

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_input_t` | struct | Bot's desired input frame: direction, speed, view angles, action flags, weapon |
| `bsp_surface_t` | struct | Surface info returned by a BSP trace (name, flags, value) |
| `bsp_trace_t` | struct | Full result of sweeping a box through the BSP world |
| `bot_entitystate_t` | struct | Snapshot of an entity's physical/animation state as seen by botlib |
| `botlib_import_t` | struct (vtable) | Engine services provided TO botlib: tracing, memory, filesystem, debug drawing |
| `aas_export_t` | struct (vtable) | AAS (Area Awareness System) functions exported by botlib |
| `ea_export_t` | struct (vtable) | Elementary Action functions — low-level client input simulation |
| `ai_export_t` | struct (vtable) | High-level AI functions: character, chat, goal, movement, weapon subsystems |
| `botlib_export_t` | struct (vtable) | Top-level export table; contains `aas`, `ea`, `ai` plus lifecycle functions |

## Global / File-Static State

None. This is a pure header.

## Key Functions / Methods

### GetBotLibAPI
- Signature: `botlib_export_t *GetBotLibAPI(int apiVersion, botlib_import_t *import)`
- Purpose: Entry point for linking the bot library; called by the engine/game to obtain the full `botlib_export_t` vtable.
- Inputs: `apiVersion` — must match `BOTLIB_API_VERSION` (2); `import` — engine-side callback table.
- Outputs/Return: Pointer to the populated `botlib_export_t` struct, or NULL on version mismatch.
- Side effects: Stores `import` pointer inside botlib for later use; initializes internal state.
- Calls: Defined in `code/botlib/be_interface.c` (not this file).
- Notes: Version guard via `BOTLIB_API_VERSION 2` macro; mismatch should produce `BLERR_LIBRARYNOTSETUP`.

> All other callable symbols are function pointers inside the vtable structs — they have no definitions here.

## Control Flow Notes
This header sits at the **init** boundary. During game startup, the server/game module calls `GetBotLibAPI` once, passing a filled `botlib_import_t`. Thereafter, per-frame calls flow through `botlib_export_t::BotLibStartFrame`, entity state updates via `BotLibUpdateEntity`, and individual bot think ticks via the `ea`/`ai` sub-tables. Shutdown calls `BotLibShutdown`. No render or client-frame involvement.

## External Dependencies
- `vec3_t`, `cplane_t`, `qboolean` — defined in `q_shared.h`
- `fileHandle_t`, `fsMode_t` — defined in `q_shared.h` / `qcommon.h`
- `pc_token_t` — defined in the botlib script/precompiler headers (`l_precomp.h`)
- Forward-declared structs (`aas_clientmove_s`, `bot_goal_s`, etc.) — defined in respective `be_aas_*.h` / `be_ai_*.h` headers
- `QDECL` calling-convention macro — defined in `q_shared.h`
