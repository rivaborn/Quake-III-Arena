# code/game/g_cmds.c — Enhanced Analysis

## Architectural Role

This file is the **command dispatcher boundary** between the server's core command system (`code/qcommon/cmd.c`) and the game module's internal state machine. Every client-issued command (except raw movement via `usercmd_t`) flows through `ClientCommand()` before reaching game logic, making it a critical choke point for server-side security, voting authority, and team/chat management. It bridges the server frame loop (which calls `vmMain(GAME_CLIENT_COMMAND)`) with the game VM's entity system and higher-level gameplay rules.

## Key Cross-References

### Incoming (who depends on this file)
- **code/server/sv_game.c**: Calls `vmMain(GAME_CLIENT_COMMAND, clientNum)` during server frame when a client command is ready
- **code/qcommon/cmd.c**: Provides `trap_Argv()`, `trap_Argc()` infrastructure that `ClientCommand` and all handlers use to read command arguments
- **code/qcommon/net_chan.c**: Indirectly — client input arrives as reliable messages that the server unpacks and queues for command processing

### Outgoing (what this file depends on)
- **code/game/g_main.c**: Calls `player_die`, `ClientUserinfoChanged`, `ClientBegin`, `CopyToBodyQue` (entity lifecycle)
- **code/game/g_team.c**: Calls `PickTeam`, `TeamCount`, `TeamLeader`, `OnSameTeam`, `CheckTeamLeader`, `SetLeader` (team logic)
- **code/game/g_utils.c**: Calls `G_LogPrintf`, `G_Printf`, `G_Error` (logging); item spawning via `BG_FindItem`, `G_Spawn`, `G_SpawnItem`, `FinishSpawningItem`, `Touch_Item`
- **code/qcommon/cvar.c** (via `trap_Cvar_VariableStringBuffer`): Reads cvars like `g_cheats`, `g_gametype`, `g_teamForceBalance` to enforce gameplay rules
- **code/ui/menudef.h**: Defines `VOICECHAT_*` constants used in voice taunt logic (compile-time dependency only)
- **Global `level` structure** (from `g_main.c`): Owns `level.voteTime`, `level.sortedClients[]`, `level.teamScores[]` — state polled every server frame

## Design Patterns & Rationale

**Command Dispatcher (Synchronous)**: `ClientCommand()` uses a linear `Q_stricmp` chain rather than a hash table. This is a trade-off for simplicity in early-2000s code; the number of commands is small (~40), so O(n) is acceptable. The dispatching is **synchronous and blocking** — no queuing.

**Handler Signature Convention**: All command handlers are `void Cmd_*_f(gentity_t *ent)`, named after Q3's command system style. This provides type safety and a discoverable naming pattern.

**State Guards (CheatsOk)**: Several handlers check `CheatsOk()` before executing, enforcing that cheats must be enabled and the player must be alive. This is **defensive in-depth** — gameplay rules are checked at the command boundary, not assumed.

**Voting as Polled State**: Unlike modern networked games that use **event-driven** voting, Q3 implements voting as mutable `level.vote*` fields that the server frame loop reads each tick (see `code/server/sv_main.c`). This simplifies implementation but couples vote logic across subsystems.

**Team Transitions (Complex State Machine)**: `SetTeam()` invokes multiple side effects — death, userinfo sync, leader recomputation — all within one function. This reflects a monolithic architecture where team changes ripple through the entire game state. Modern engines would decompose this into separate systems (death → body queue → respawn).

## Data Flow Through This File

```
Client UDP packet arrives
    ↓
code/server/sv_client.c unpacks reliable message
    ↓
vmMain(GAME_CLIENT_COMMAND, clientNum) called
    ↓
ClientCommand(clientNum) reads command string via trap_Argv(0, ...)
    ↓
Q_stricmp dispatch to Cmd_*_f handler
    ↓
Handler reads args via trap_Argv(1..n, ...)
    ↓
[Cheat check via CheatsOk() if required]
    ↓
Game state mutation (entity flags, client session, level.vote*, etc.)
    ↓
Output via trap_SendServerCommand() → code/server/sv_snapshot.c → clients
```

**Key state mutations**:
- `entity->flags` (godmode, notarget, etc. for cheats)
- `client->sess.sessionTeam` (team changes trigger cascading updates)
- `level.vote*` (voting state polled by server frame loop each `sv_fps`)
- `level.clients[].ps.stats[]`, `level.clients[].ps.ammo[]` (cheat commands)

## Learning Notes

**Era-appropriate design**: This code reflects **2003 Quake III engine style** — synchronous command dispatch, global mutable state, minimal abstraction. Modern engines use **message queues**, **event buses**, or **command objects** to decouple timing and allow async/replay semantics.

**Boundary clarity**: The `trap_*` syscall boundary is very clean and explicit. Every interaction with the engine goes through indexed syscalls. This made it possible to sandbox the game module and recompile it without relinking the engine.

**Data-driven attempts**: The `gc_orders[]` static array and voice taunt context-sensitivity show early moves toward **table-driven behavior**, but it's still hardcoded — a later evolution would move these to `.cfg` files or `.asm` asset tables.

**Team system complexity**: `SetTeam()` reveals that Q3's team/spectator mechanics are more intricate than they appear — tournament mode locking, balance enforcement, leader assignment, and follow-mode spectating all live here. This is a good example of **business logic at the command boundary** rather than in a higher-level rules engine.

**What's missing**: No undo/transaction semantics. If `SetTeam()` crashes mid-transition, the game state is inconsistent. No command validation schema — parameters are parsed ad-hoc by each handler. No rate limiting at this layer (handled separately in `code/server/sv_client.c`).

## Potential Issues

- **Buffer overflow in `DeathmatchScoreboardMessage`**: Uses `strcpy` with a loop-based length check (`stringlength + j > 1024`). If `entry` is exactly 1024 bytes, the check may not prevent overflow into the next frame's stack.
- **Silent truncation in `ConcatArgs`**: Silently truncates arguments if total length exceeds `MAX_STRING_CHARS`. Could cause vote parameters or chat messages to be silently cut off without warning.
- **Injection risk in voting**: `Cmd_CallVote_f` rejects semicolons to prevent command injection, but this is a **blacklist approach** — a malicious client might find other injection vectors. A whitelist of allowed vote types would be more robust.
- **Race condition potential**: If a client issues multiple rapid team-change commands, the `SetTeam()` function could be called concurrently (in listen-server mode where client and server run in the same process). Not exploitable in standard server mode, but possible in editor/testing scenarios.
