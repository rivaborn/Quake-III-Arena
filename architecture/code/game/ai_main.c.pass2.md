# code/game/ai_main.c — Enhanced Analysis

## Architectural Role

`ai_main.c` is the **Game VM's sole orchestration hub for the entire bot AI pipeline**. It sits precisely at the intersection of three subsystems: the server-side Game VM (which owns game state), the botlib (which owns navigation and AI decision logic), and the engine's syscall layer (which owns collision, entity linking, and network commands). Every frame, it pulls entity state from the game world, drives the botlib update, triggers per-bot AI ticks, and converts botlib output back into `usercmd_t` commands indistinguishable from human input — completing a full sense-think-act loop within the QVM sandbox.

Its `BotAI_*` helper functions (`BotAI_Trace`, `BotAI_GetEntityState`, `BotAI_GetSnapshotEntity`) serve as **registered callbacks** in the `botlib_import_t` vtable built by `BotInitLibrary`. The botlib never holds a direct reference to any game symbol; it calls back into the game exclusively through these function pointers. This bidirectional indirection is what allows botlib to be a separate, reusable binary component.

---

## Key Cross-References

### Incoming (who depends on this file)

- **`code/game/g_bot.c`**: Calls `BotAISetup`, `BotAIShutdown`, `BotAILoadMap`, `BotAIStartFrame`, `BotAISetupClient`, `BotAIShutdownClient`. This is the primary driver — `g_bot.c` handles bot spawning/removal from the game side, while `ai_main.c` handles the AI lifecycle.
- **`code/game/g_main.c`**: Indirectly drives `BotAIStartFrame` each server frame via `G_RunFrame` → `g_bot.c`.
- **`code/game/ai_dmq3.c`, `ai_dmnet.c`, `ai_team.c`, `ai_chat.c`, `ai_cmd.c`**: All read the global `botstates[]` array and `floattime`/`numbots` defined here. These sibling AI files implement higher-level behavior but depend on `ai_main.c` for the shared bot state context.
- **`code/botlib/` (via vtable)**: The botlib calls `BotAI_Trace`, `BotAI_GetClientState`, `BotAI_GetEntityState`, `BotAI_GetSnapshotEntity`, and `BotAI_Print` through function pointers registered in `botlib_import_t`. These are the only game→botlib "upstream" callbacks.

### Outgoing (what this file depends on)

- **`code/botlib/` via `trap_BotLib*` syscalls** (opcode range 200–599): Every botlib call (`trap_BotAllocGoalState`, `trap_BotLibStartFrame`, `trap_BotLibUpdateEntity`, `trap_EA_GetInput`, `trap_EA_View`, etc.) crosses the VM boundary through `g_syscalls.c`. `ai_main.c` never links to botlib symbols directly.
- **`code/game/g_local.h` / `g_entities[]`**: Reads entity state (`g_entities[clientNum]`, `ent->client->ps`, `ent->s`, `ent->r`) in the adapter callbacks to translate game-world information into botlib-compatible structs.
- **`code/game/ai_dmq3.c`**: Calls `BotDeathmatchAI` (the FSM driver) and `BotSetupDeathmatchAI` from `BotAI` and `BotAILoadMap`.
- **`code/game/ai_chat.c`, `ai_cmd.c`, `ai_dmnet.c`, `ai_vcmd.c`**: Called from `BotAIShutdownClient` (`BotChat_ExitGame`, `BotFreeWaypoints`, `BotClearActivateGoalStack`) and `BotAI` (`BotVoiceChatCommand`, server command processing).
- **`code/qcommon` (AAS/CM) via `trap_AAS_*`, `trap_Trace`**: `BotAI_Trace` wraps `trap_Trace`; `BotAIStartFrame` calls `trap_AAS_Time`; `BotAILoadMap` calls `trap_BotLibLoadMap`.

---

## Design Patterns & Rationale

**Adapter / Translation Layer**: The `BotAI_*` callback family exists solely to translate between two incompatible type systems. The botlib uses `bsp_trace_t`, `bot_entitystate_t`, `entityState_t`; the game VM uses `trace_t`, `gentity_t`, `playerState_t`. The adapter layer ensures neither side is contaminated by the other's types, preserving modularity for potential non-Quake3 botlib reuse.

**Inversion of Control via Function-Pointer Vtable**: The `botlib_import_t` registered during `BotInitLibrary` inverts the normal dependency direction. The botlib is a consumer of services it never names explicitly, enabling offline (bspc) and online (game) use of the same botlib codebase with entirely different service implementations.

**Residual-Based Staggered Scheduling**: Rather than thinking every frame, bots use per-bot `botthink_residual` accumulators and the global `bot_thinktime` cvar to throttle AI ticks. This is a deliberate CPU budget pattern: at `bot_thinktime=100ms`, 8 bots at 20Hz server rate each think once every 2 frames on average, staggered so no two bots think on the same frame. Modern engines handle this with job queues; here it's manual residual arithmetic.

**Genetic Algorithm for Fuzzy Logic**: `BotInterbreedBots`/`BotInterbreeding` implement a rudimentary evolutionary system. K/D ratio serves as fitness; `trap_GeneticParentsAndChildSelection` selects parents; `trap_BotInterbreedGoalFuzzyLogic` performs crossover; `trap_BotMutateGoalFuzzyLogic` adds noise. This was a forward-looking feature in 1999 — auto-tuning bot "personalities" from match performance — but it operates on in-memory fuzzy weight tables that aren't exposed to level designers.

**Tradeoff — Global Mutable Array**: `botstates[]` is a global array of raw pointers. This avoids any indirection overhead (critical for a 1999 engine) but means all bot AI files share mutable state with no protection. Any file that includes `ai_main.h` can corrupt any bot's state.

---

## Data Flow Through This File

```
Server frame tick
    └─► BotAIStartFrame(time)
            │
            ├─► trap_BotLibStartFrame()          [clock botlib AAS time]
            ├─► trap_BotLibUpdateEntity() × N    [push all entity states into botlib]
            │       data: g_entities[i].s → bot_entitystate_t (type, origin, angles, etc.)
            │
            ├─► BotAI(client, thinktime)          [per-bot, throttled by residual]
            │       │
            │       ├─► BotAI_GetClientState()    [pull ps from g_entities]
            │       ├─► trap_BotGetServerCommand() [dequeue server→bot messages]
            │       ├─► BotDeathmatchAI()          [FSM: goals → EA commands queued in botlib]
            │       └─► trap_EA_SelectWeapon()
            │
            └─► BotUpdateInput(bs, time, elapsed) [every bot, every frame]
                    │
                    ├─► BotChangeViewAngles()      [smooth view angle toward ideal]
                    │       └─► trap_EA_View()     [write to EA input buffer]
                    ├─► trap_EA_GetInput()         [retrieve bot_input_t from EA layer]
                    ├─► BotInputToUserCommand()    [bot_input_t → usercmd_t]
                    └─► trap_BotUserCommand()      [inject usercmd into server]
```

Key state transitions:
- `bot_input_t.speed` [0–400] → scaled to `usercmd_t.forwardmove/rightmove` [0–127]
- View angles: `bs->ideal_viewangles` (set by `BotDeathmatchAI`) → smoothed via `BotChangeViewAngles` → `bs->viewangles` → written to EA layer → recovered in `BotInputToUserCommand` as `ucmd->angles[YAW/PITCH]`
- `bs->areanum`: Updated each tick via `BotPointAreaNum(bs->origin)` — this is the AAS area the bot occupies, the fundamental currency for all routing queries

---

## Learning Notes

**The sense-think-act loop is explicit and manual**: Modern game AI often uses behavior trees or ECS job graphs; here the full loop is a hand-written `BotAIStartFrame` function you can read top-to-bottom. The staggering, the entity update pump, the usercmd injection — all visible in one place.

**Bots ARE clients**: `BotUpdateInput` ends with `trap_BotUserCommand(bs->client, &ucmd)`. The server processes this usercmd identically to a real player's input. Bots go through the same `Pmove`, hit detection, and scoring as humans. This is the cleanest possible architecture for a multiplayer game bot: no special-casing in physics or combat.

**QVM sandboxing**: The game VM cannot call botlib functions directly — they cross the syscall boundary via `trap_BotLib*`. This means the entire `code/botlib/` tree could be replaced with a different navigation system without recompiling the game VM, as long as the ABI (opcode numbers, struct layouts) is preserved.

**`vsprintf` to a 2048-byte stack buffer** in `BotAI_Print` is a classic late-1990s pattern. Any format string producing >2048 bytes silently stack-smashes. Modern engines use `Q_vsnprintf` with an explicit bound.

**Interbreeding is tournament-only**: `BotInterbreeding` only activates when `numplayers == 1` (single human, rest bots) and requires explicit cvar setup. It was never surfaced as a runtime feature in shipped Q3A — it's research code that shipped in the release build.

---

## Potential Issues

- **`vsprintf` overflow** in `BotAI_Print` (line ~90): unbounded format into a 2048-byte stack buffer; any bot chat string exceeding that length is a stack overwrite. Real exploit surface is low since format strings come from `.char` files, but it's worth noting.
- **`botstates[]` is never bounds-checked** against `MAX_CLIENTS` consistently — several loops use `maxclients` (runtime value) clamped against `MAX_CLIENTS` (compile-time). If `maxclients` ever exceeds `MAX_CLIENTS` at runtime (server misconfiguration), the loop bound is correct but the array declaration is the real constraint.
- **`regularupdate_time` global**: Shared across all bots with no per-bot equivalent, meaning `BotAIRegularUpdate` fires for all bots simultaneously every 0.3 s rather than being staggered like `BotAI`. If `BotAIRegularUpdate` is ever expensive (it checks team state), this creates a periodic CPU spike.
