# Subsystem Overview

## Purpose
The `code/game` subsystem is the server-side game VM for Quake III Arena, executing as a QVM bytecode module loaded by the engine. It owns all authoritative game logic: entity simulation, player physics, combat, item management, team/CTF rules, and the full bot AI stack — translating bot decisions and player inputs into server-state mutations each frame.

## Key Files

| File | Role |
|---|---|
| `g_local.h` | Central private header; defines `gentity_t`, `gclient_t`, `level_locals_t`, all cross-file function declarations, cvar globals, and the full `trap_*` syscall interface |
| `g_main.c` | VM entry point (`vmMain`); owns the frame loop, cvar table, level init/shutdown, voting, and score ranking |
| `g_active.c` | Per-client per-frame driver: runs `Pmove`, applies environmental damage, dispatches events, synchronizes `playerState_t` → `entityState_t` |
| `g_client.c` | Full client lifecycle: connect, spawn, respawn, userinfo, body queue, disconnect, spawn-point selection |
| `g_combat.c` | Central damage pipeline: `G_Damage`, `G_RadiusDamage`, death sequencing, scoring, item drops |
| `g_weapon.c` | Weapon fire: hitscan traces and projectile spawning for all weapons; accuracy tracking |
| `g_missile.c` | Missile entity movement, collision, bounce, impact, and prox-mine lifecycle |
| `g_items.c` | Item pickup logic, respawn timers, world spawn, dynamic launch/drop, per-frame physics |
| `g_team.c` | All CTF/team logic: flag lifecycle, frag bonuses, obelisk/harvester mechanics, team spawn selection |
| `g_spawn.c` | BSP entity string parser and class-dispatch; entry point for all entity instantiation at level load |
| `g_mover.c` | Moving brush entities: doors, platforms, buttons, trains, push/block collision |
| `g_trigger.c` | Volume triggers: jump pads, teleporters, hurt zones, repeating timers |
| `g_target.c` | Target entities: message printers, audio speakers, damage lasers, score modifiers, teleport destinations |
| `g_misc.c` | Teleportation, portal surfaces, weapon shooters, utility map entities |
| `g_bot.c` | Bot lifecycle: parse bot/arena configs, add/remove bots, enforce `bot_minplayers`, stagger spawns |
| `g_utils.c` | Core entity utilities: spawn/free/temp entities, search, targeting, event signaling, shader remapping |
| `g_mem.c` | Fixed 256 KB bump-pointer allocator; no free, reset per map |
| `g_syscalls.c` | DLL-side syscall wrappers; bridges typed C calls to the engine's variadic `syscall` dispatch (excluded from QVM builds) |
| `g_cmds.c` | Client command dispatcher: chat, team management, voting, cheat commands, scoreboard |
| `g_svcmds.c` | Server-console command dispatcher: IP ban management, entity listing, admin commands |
| `g_session.c` | Serializes/deserializes per-client session state to cvars across level loads |
| `g_arenas.c` | Intermission podium: collect end-match stats, spawn victory podium entities, drive winner animations |
| `g_rankings.c` | Online rankings integration: submit per-player stats and match metadata to an external ranking service |
| `ai_main.c` | Bot AI hub: botlib init/shutdown, per-frame `BotAIStartFrame`, `usercmd_t` synthesis, entity state feed, interbreeding |
| `ai_dmq3.c` | Core deathmatch AI tick: enemy detection, aim prediction, team goal selection, inventory updates, event processing |
| `ai_dmnet.c` | Bot FSM: all `AIEnter_*` state transitions and `AINode_*` per-frame state handlers |
| `ai_team.c` | Team leadership: elect leader, sort teammates by travel time, issue role-assignment orders |
| `ai_chat.c` | Bot chat layer: rate-limiting, position validation, category selection, chat delegation |
| `ai_cmd.c` | Chat/voice command parser: maps natural-language team commands to `bot_state_t` LTG changes |
| `ai_vcmd.c` | Voice command handler: maps incoming `VOICECHAT_*` strings to bot behavioral state changes |
| `bg_pmove.c` | Deterministic player movement simulation (`Pmove`); shared with cgame for client-side prediction |
| `bg_slidemove.c` | Sliding collision response and step-up logic; called from `bg_pmove.c` |
| `bg_misc.c` | Shared item registry (`bg_itemlist`), trajectory evaluation, `playerState_t` ↔ `entityState_t` conversion |
| `bg_lib.c` | QVM-only libc replacement: `qsort`, string/math/print functions |
| `q_shared.c` | Universal utility library: string handling, parsing, byte-order, info-string manipulation |
| `q_math.c` | Stateless 3D math library: vectors, angles, planes, bounding boxes, fast approximations |
| `g_local.h` / `bg_public.h` / `q_shared.h` | Header hierarchy establishing the shared type contract across the subsystem |
| `chars.h` / `inv.h` / `syn.h` / `match.h` | Bot AI constant tables: personality indices, inventory slots, chat context flags, message type codes |
| `g_rankings.h` | `QGR_KEY_*` constants encoding stat semantics for the online rankings backend |

## Core Responsibilities

- **Entity simulation**: Spawn, think, move, and free all server-side entities each frame (`G_RunFrame`), including movers, missiles, items, triggers, and targets
- **Player physics and state**: Run `Pmove` authoritatively for each client, apply environmental damage, synchronize `playerState_t` → `entityState_t`, and dispatch events to other clients
- **Combat pipeline**: Route all damage (hitscan, splash, hazard) through `G_Damage`, apply armor/protection rules, execute death sequencing, manage scoring and item drops
- **Team and CTF rules**: Maintain flag state, award team bonuses, manage obelisk/harvester objectives, enforce team spawn placement, and broadcast team events
- **Bot AI stack**: Drive a full per-bot FSM (Seek/Battle/Respawn/Observer states), translate bot decisions into `usercmd_t` inputs via the botlib EA layer, manage bot lifecycle and interbreeding across map sessions
- **Level loading and entity instantiation**: Parse BSP entity strings, dispatch class-specific spawn functions, and initialize the complete world state at map load
- **Client lifecycle management**: Handle connect, spawn, userinfo, session persistence, body queue, and disconnect sequences
- **VM/engine boundary**: Expose `vmMain` as the sole engine entry point; bridge all engine services through typed `trap_*` wrappers; share physics and item data with the cgame module via the `bg_*` layer

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `vmMain` — single export to the engine; receives `GAME_INIT`, `GAME_RUN_FRAME`, `GAME_CLIENT_CONNECT`, `GAME_CLIENT_COMMAND`, etc. via `gameExport_t`
- `sharedEntity_t` / `entityShared_t` — memory layout read directly by the server engine for entity broadcasting
- `playerState_t` fields within `gclient_t` — readable by the engine for snapshot generation
- `bg_pmove.c` / `bg_misc.c` (`Pmove`, `BG_PlayerStateToEntityState`, `bg_itemlist`, trajectory functions) — shared with the `cgame` module for client-side prediction; must remain deterministically identical
- Config-string writes via `trap_SetConfigstring` — communicate level state (items, teams, scores) to clients

**Consumes from other subsystems:**
- **Engine** (via `trap_*` syscalls): collision traces (`trap_Trace`), entity linking (`trap_LinkEntity`), cvar access, filesystem, server commands, AAS/botlib dispatch (opcodes 200–599)
- **Botlib** (`botlib_export_t`): AAS navigation queries, elementary actions (`EA_*`), goal/move/weapon/chat AI APIs — called through the `trap_BotLib*` syscall range
- **BSP/collision** (`cm_*`): indirectly consumed through `trap_Trace`, `trap_PointContents`, `trap_SetBrushModel`
- **`q_shared.h` / `q_math.c` / `q_shared.c`**: foundational types, math, and string utilities compiled into the game VM

## Runtime Role

**Init (`GAME_INIT` → `G_InitGame`):**
1. Register and update all cvars (`gameCvarTable`)
2. Initialize the entity array and level state (`G_InitMemory`, `G_ResetEntities`)
3. Spawn all map entities from the BSP entity string (`G_SpawnEntitiesFromString`)
4. Initialize the botlib and load AAS data (`BotAISetup`, `BotAILoadMap`)
5. Validate team items (flags, obelisks) and set initial config strings

**Frame (`GAME_RUN_FRAME` → `G_RunFrame`):**
1. Process queued client commands and think for all active entities
2. Run `Pmove` for each client via `ClientThink` → `G_active.c`
3. Advance missiles, movers, items, and trigger volumes
4. Drive bot AI: `BotAIStartFrame` → per-bot `BotAI` → `BotDeathmatchAI` → FSM node → EA input → `usercmd_t` submission
5. Synchronize entity state, dispatch events, update scores and rankings

**Shutdown (`GAME_SHUTDOWN` → `G_ShutdownGame`):**
1. Serialize all client session data to cvars
2. Shut down botlib and free per-bot AI state (`BotAIShutdown`)
3. Submit final match statistics to rankings service (`g_rankings.c`)

## Notable Implementation Details

- **QVM dual-compile boundary**: Files prefixed `bg_` (`bg_pmove.c`, `bg_misc.c`, `bg_lib.c`) compile into both the game VM and the cgame VM to guarantee identical physics simulation for client-side prediction. `g_syscalls.c` is excluded from QVM builds; `g_syscalls.asm` is used instead.
- **Bot state machine**: `bot_state_t` (defined in `ai_main.h`) is a monolithic per-bot record spanning FSM node pointer, LTG/NBG goals, combat state, team/CTF tracking, waypoint chains, and session data. The FSM is driven by function pointers (`bs->ainode`) set by `AIEnter_*` functions and invoked each frame by `BotAI`.
- **Botlib boundary**: The game module never links directly to botlib object code; all botlib calls go through the `trap_BotLib*` syscall range (opcodes 200–599), keeping botlib loadable as a separate shared library with its own `botlib_import_t` / `botlib_export_t` vtable pair.
- **`bg_itemlist` / `inv.h` coupling**: `MODELINDEX_*` constants in `inv.h` must stay manually synchronized with the ordering of `bg_itemlist[]` in `bg_misc.c`; a mismatch silently corrupts bot item recognition with no compile-time check.
- **Fixed-size game allocator**: `g_mem.c` provides a 256 KB bump-pointer pool with no free operation; all game-side dynamic allocations (bot info strings, arena records, waypoints) are permanent for the map session lifetime.
- **`MISSIONPACK` conditional compilation**: A significant fraction of the subsystem's behavior (Team Arena weapons, obelisk/harvester game modes, prox mines, Kamikaze, voice chat dispatch, alternate scoring constants) is gated behind `#ifdef MISSIONPACK`, producing two distinct binary configurations from a single source tree.
- **Chat matching system**: `ai_cmd.c` uses `trap_BotFindMatch` with template patterns from `match.h` to parse free-text team chat into structured command dispatches, bridging natural-language player input and the bot's internal LTG state without a grammar parser.
