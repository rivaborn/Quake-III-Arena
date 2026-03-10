# code/game/ai_main.h

## File Purpose
Central header for Quake III Arena's in-game bot AI system. Defines the monolithic `bot_state_t` structure that tracks all per-bot runtime state, along with constants for bot behavior flags, long-term goal types, team/CTF strategies, and shared AI utility function declarations.

## Core Responsibilities
- Define all bot behavioral flag constants (`BFL_*`) and long-term goal type constants (`LTG_*`)
- Define goal dedication timeouts for team and CTF scenarios
- Declare `bot_state_t`, the master per-bot state record spanning movement, goals, combat, team, and CTF data
- Declare `bot_waypoint_t` for checkpoint/patrol point linked lists
- Declare `bot_activategoal_t` for a stack of interactive object activation goals
- Expose the `FloatTime()` macro and utility function declarations used across AI subsystems

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_waypoint_t` | struct | Linked-list node for named waypoints used in checkpoint/patrol routing |
| `bot_activategoal_t` | struct | Stacked goal record for activating map triggers/buttons; tracks shoot target, blocked routing areas |
| `bot_state_t` | struct | Master bot runtime state: player state, AI node pointer, combat, movement, team/CTF, formation, activation stack, patrol |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `floattime` | `float` | global (extern) | Shared floating-point game time; accessed via `FloatTime()` macro across all AI files |

## Key Functions / Methods

### BotResetState
- Signature: `void BotResetState(bot_state_t *bs)`
- Purpose: Zero/reinitialize all fields of a bot's state, called on spawn or map restart
- Inputs: Pointer to the bot state to reset
- Outputs/Return: void
- Side effects: Modifies `*bs` in place
- Calls: Defined in `ai_main.c`; not inferable from this file
- Notes: Must be called before the bot begins thinking to avoid stale state

### NumBots
- Signature: `int NumBots(void)`
- Purpose: Returns the count of active bots currently in the game
- Inputs: None
- Outputs/Return: Integer bot count
- Side effects: None
- Calls: Defined in `ai_main.c`
- Notes: Used by team AI logic to determine team composition

### BotEntityInfo
- Signature: `void BotEntityInfo(int entnum, aas_entityinfo_t *info)`
- Purpose: Fills an `aas_entityinfo_t` with AAS-level data for a given entity number
- Inputs: Entity number; output struct pointer
- Outputs/Return: void (result via out-param)
- Side effects: None
- Calls: Wraps botlib AAS entity queries; defined in `ai_main.c`
- Notes: Bridges the game-side bot code to the botlib AAS layer

### BotAI_Print
- Signature: `void QDECL BotAI_Print(int type, char *fmt, ...)`
- Purpose: Varargs print/logging function for bot AI debug and error output
- Inputs: Message type enum, printf-style format string and args
- Outputs/Return: void
- Side effects: I/O to console or log
- Calls: Likely wraps `trap_Printf` or `Com_Printf`; defined in game source

### BotAI_Trace
- Signature: `void BotAI_Trace(bsp_trace_t *bsptrace, vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int passent, int contentmask)`
- Purpose: Performs a BSP trace on behalf of bot AI code, bridging botlib trace calls to the game's CM system
- Inputs: Output trace struct, start/end points, bounding box, entity to pass through, content mask
- Outputs/Return: void (result via `*bsptrace`)
- Side effects: None beyond populating the trace result
- Calls: Wraps `trap_Trace`; defined in game source

**Notes on remaining declarations:** `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotAI_GetEntityState`, `BotAI_GetSnapshotEntity`, and `BotTeamLeader` are thin game-side wrappers over trap calls or server-query functions, bridging botlib callbacks into the VM game module.

## Control Flow Notes
This header is included by all `ai_*.c` files. The `ainode` function pointer in `bot_state_t` is the per-frame AI dispatch mechanism — each frame, `ai_main.c` calls `bs->ainode(bs)`, which routes to a behavior state such as combat, roaming, or CTF logic. `FloatTime()` is used ubiquitously for time comparisons against the many `*_time` fields in `bot_state_t`.

## External Dependencies
- `bg_public.h` / game headers: `playerState_t`, `usercmd_t`, `entityState_t`, `vec3_t`
- `botlib.h` / `be_aas.h`: `bot_goal_t`, `bot_settings_t`, `aas_entityinfo_t`, `bsp_trace_t`
- `be_ai_move.h`: move state handle type (used in `ms` field)
- Trap functions (`trap_Trace`, `trap_Printf`, etc.) — defined in `g_syscalls.c`, called from game VM
