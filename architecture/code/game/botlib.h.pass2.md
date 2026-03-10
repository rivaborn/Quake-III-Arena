# code/game/botlib.h — Enhanced Analysis

## Architectural Role

This header defines the **versioned, two-way dynamic interface contract** between the game module and the runtime-loaded botlib library. The server `sv_bot.c` populates a `botlib_import_t` vtable with engine callbacks (tracing, memory, FS, debug viz) and calls `GetBotLibAPI(BOTLIB_API_VERSION, &botlib_import)` to obtain a `botlib_export_t` vtable. Thereafter, the game VM invokes botlib via `trap_BotLib*` syscall range (opcodes 200–599), which dispatch to pointers in the returned export table. This creates a strictly encapsulated subsystem that never directly sees BSP or raw game entities—only precompiled AAS graphs and snapshot `bot_entitystate_t` updates.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/server/sv_bot.c`**: Drives per-frame bot AI ticks; populates `botlib_import_t` at server startup; calls `BotLibStartFrame`, `BotLibUpdateEntity` each frame
- **`code/game/g_bot.c`** (game VM): Spawns/despawns bot entities; calls `trap_BotLib*` syscalls (200–599 range) mapped to `botlib_export_t` entries via `SV_GameSystemCalls`
- **`code/game/ai_main.c`** (game VM): Per-bot FSM; calls `trap_BotLibAI*` (goal, move, chat, weapon subsystems) to synthesize `usercmd_t`
- **`code/botlib/be_interface.c`**: Defines `GetBotLibAPI` implementation (not in this header, but entry point for dynamic linking)

### Outgoing (what this file depends on)
- **`code/qcommon/qcommon.h`**: Provides `vec3_t`, `cplane_t`, `qboolean`, `fileHandle_t`, `fsMode_t` base types
- **`code/botlib/l_precomp.h`** (implied): Forward-declares `pc_token_t` (script preprocessor tokens)
- **`code/bspc/aasfile.h`** (offline tool): Defines AAS binary file format constants/structures; runtime botlib loads precompiled `.aas` files
- **Renderer debug**: `botlib_import_t` includes `DebugLineCreate/Show/Delete` and `DebugPolygonCreate/Delete` callbacks for visualization

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Versioned Plugin Interface** | `BOTLIB_API_VERSION 2` guard; `GetBotLibAPI(int apiVersion, ...)` | Decouple botlib binary versioning from game; prevent ABI mismatches when either side updates |
| **Dependency Injection** | `botlib_import_t` passed at init, captured internally | Engine services (traces, memory, FS) stay in qcommon/server; botlib remains platform-agnostic |
| **Vtable Composition** | `botlib_export_t` contains nested `aas_export_t`, `ea_export_t`, `ai_export_t` | Organize 100+ bot functions into logical subsystems without creating separate loader entries |
| **Action Flags (Bitmask)** | `#define ACTION_ATTACK 0x0000001`, `ACTION_JUMP 0x0000010`, etc. | Pack bot's per-frame command intent into a single int for efficient serialization in `bot_input_t` |
| **State Snapshots** | `bot_entitystate_t` mirrors game `entityState_t` (origin, angles, animation frames, powerups) | Immutable snapshots allow botlib to predict movement without seeing live entity mutations |
| **Two-Phase Lifecycle** | `BotLibSetup` → frame-by-frame `BotLibStartFrame`/`BotLibUpdateEntity` → `BotLibShutdown` | Lazy initialization, per-map AAS loading, graceful teardown |

## Data Flow Through This File

```
┌─────────────────────────────────────────────────────────────┐
│ Game Startup (sv_bot.c)                                      │
│ ┌──────────────────────────────────────────────────────────┐│
│ │ Populate botlib_import_t (traces, mem, FS, debug)       ││
│ │ Call GetBotLibAPI(BOTLIB_API_VERSION, &botlib_import)   ││
│ │ → Returns botlib_export_t vtable (or NULL on mismatch)  ││
│ └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Per-Map Load (BotLibLoadMap)                                │
│ ┌──────────────────────────────────────────────────────────┐│
│ │ botlib reads BSP, loads AAS file (via botlib_import.FS) ││
│ │ Initializes AAS graph, routing caches, entity index     ││
│ └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Per-Frame (SV_Frame → sv_bot.c)                             │
│ ┌──────────────────────────────────────────────────────────┐│
│ │ 1. BotLibStartFrame(frameTime)                           ││
│ │ 2. BotLibUpdateEntity(bot_num, bot_entitystate_t)       ││
│ │    → botlib re-links bot to AAS areas                    ││
│ │ 3. Calls (via game VM syscalls):                         ││
│ │    - AAS_PredictRoute (path planning)                    ││
│ │    - BotChooseLTGItem / BotChooseNBGItem (goal select)   ││
│ │    - BotMoveToGoal (FSM: step, jump, climb)             ││
│ │    - BotChooseBestFightWeapon (inventory eval)          ││
│ │    - BotGetChatMessage (NPC dialogue)                    ││
│ │ 4. EA_GetInput(bot_num, thinktime) → accumulates        ││
│ │    bot_input_t (dir, speed, viewangles, actions, wpn)   ││
│ │ 5. game VM converts bot_input_t → usercmd_t             ││
│ └──────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
```

## Learning Notes

**What this file teaches:**
- **Game AI at scale (2005 era)**: Encapsulated AI library with offline preprocessing (AAS compiler). Modern engines use runtime nav-meshes or learned policies; Q3 chose static, precomputed area graphs.
- **Botlib versioning strategy**: Single `BOTLIB_API_VERSION` constant guards both sides; updating either game or botlib requires mutual consent. No semantic versioning—binary-level ABI contract.
- **Action encoding**: `bot_input_t` and `ACTION_*` flags show how 2000s-era games packed control input into compact structs for determinism and replays (compare modern event-based input systems).
- **Callback-driven physics**: Botlib never owns collision or entity state—always asks qcommon/server via `botlib_import_t::Trace`, `PointContents`, `inPVS`. This is classical plugin architecture.
- **Personality via data**: `be_ai_char.h` abstracts bot skill/behavior into fuzzy logic weights loaded from text files; modern engines tend toward behavior trees or planners.

**Idiomatic to Quake III / different from modern engines:**
- **Manual memory management**: No allocators; raw `GetMemory`/`FreeMemory` from zone
- **Error codes instead of exceptions**: `BLERR_*` constants; caller must check return values
- **Stateful action accumulation**: `EA_*` functions build up state; `EA_EndRegular` flushes one frame
- **Script-based configuration**: Library variables (phys_friction, rs_jumppad, etc.) tuned via console at runtime
- **Static AAS graphs**: No dynamic obstacles or runtime nav-mesh generation

## Potential Issues

- **Silent initialization failure**: `GetBotLibAPI` returns NULL on version mismatch, but the header does not show error handling or fallback—caller must check and decide what to do (likely calls `Com_Error`).
- **Lifecycle coupling**: `BotLibSetup` must be called before any other export; no runtime checks prevent misuse (e.g., calling `AAS_*` before map load). State machine is implicit.
- **Memory leak risk**: `botlib_import_t::GetMemory`/`FreeMemory` are unpaired in the header; caller (game VM) is responsible for correct pairing, but no RAII or scope guards.
- **No thread-safety**: No explicit locks or atomics in vtable; single-threaded operation assumed (qcommon has no mutex infrastructure visible here).
- **Action flag collisions**: Several flags unused (0x0000004, 0x0000040, etc.); no documentation of why; future extensions risk bitfield overflow.
