# code/botlib/be_ea.c — Enhanced Analysis

## Architectural Role

`be_ea.c` implements the **command accumulator** interface between botlib's high-level AI decision modules and the game engine's input system. It sits at the lowest level of the bot AI stack, receiving calls from all botlib AI modules (`be_ai_move.c`, `be_ai_goal.c`, `be_ai_weap.c`, `be_ai_chat.c`) and translating them into a unified `bot_input_t` state buffer that the game VM will submit to the engine each frame. This is the final bottleneck through which all bot behavior must pass to affect gameplay.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/botlib/be_ai_move.c`**: Calls `EA_Move`, `EA_View`, `EA_Jump`, `EA_Crouch`, `EA_Walk` to execute movement decisions
- **`code/botlib/be_ai_weap.c`**: Calls `EA_SelectWeapon`, `EA_Attack` to manage firing
- **`code/botlib/be_ai_chat.c`**: Calls `EA_Say`, `EA_SayTeam`, `EA_Tell` for voice communication
- **`code/botlib/be_ai_gen.c`** (implied): Likely calls various action setters to express generic bot behaviors
- **`code/game/g_bot.c`** (server-side game VM): Calls `EA_Setup` during botlib initialization and implicitly depends on `EA_GetInput`/`EA_ResetInput` via the main botlib frame loop

### Outgoing (what this file depends on)

- **`botimport.BotClientCommand`** (engine callback from `botlib_import_t`): For text-based commands (`say`, `say_team`, `tell`, `use`, `drop`, `invuse`, `invdrop`)
- **`botlibglobals.maxclients`** (from `be_interface.c`): Determines allocation size for `botinputs` array
- **`GetClearedHunkMemory`** (from `l_memory.h`): Allocates the hunk-resident `botinputs` array
- **`FreeMemory`** (from `l_memory.h`): Deallocates on shutdown
- **`Com_Memcpy`, `VectorCopy`, `VectorClear`** (from `q_shared.h`): Low-level utilities for state copying

## Design Patterns & Rationale

**Command Accumulator Pattern**: Rather than immediately executing each action (which would fragment state across multiple frames), `be_ea.c` buffers all desired actions into a single `bot_input_t` state object. This allows the AI to issue multiple conflicting commands in a single think cycle (e.g., both `EA_Jump` and movement), and the game engine decides the final precedence. This is exactly how human player input works in Q3: the client collects all button presses and view angles into one `usercmd_t` per frame, not one per event.

**Jump De-bounce via State Carryover**: The `ACTION_JUMPEDLASTFRAME` flag carries jump state across frame boundaries. This prevents jump triggers from re-firing on consecutive frames—a human player must release and re-press the jump key. The pattern shows careful attention to mimicking human input realism.

**Global Per-Client Array**: A single flat `botinputs[MAX_CLIENTS]` array indexed by client ID is typical for Q3's fixed-size client slot model. This avoids heap fragmentation and keeps client state cache-coherent. The design assumes bots occupy regular client slots (0–N) rather than a separate pool.

**Deferred Reset**: `EA_ResetInput` clears state only *after* it's been snapshotted via `EA_GetInput`. This is a critical phase-ordering requirement: snapshot happens first (for this frame's game simulation), reset happens second (preparing for next frame's AI decisions).

## Data Flow Through This File

```
Per-frame AI think cycle:
  [Higher-level AI: be_ai_move.c, be_ai_weap.c, etc.]
         ↓ (calls EA_Move, EA_Attack, EA_SelectWeapon, etc.)
  [botinputs[client] state accumulation]
         ↓ (at end of think)
  EA_GetInput(client, ...) → snapshot to output buffer
         ↓ (game VM submits to engine)
  Engine applies usercmd
         ↓ (next frame prep)
  EA_ResetInput(client) → clear for next frame
         ↓
  [Next frame's AI think cycle begins]
```

The jump de-bounce state carries forward: if `ACTION_JUMP` was set this frame, `ACTION_JUMPEDLASTFRAME` is set for next frame's `EA_Jump` call to see and suppress.

## Learning Notes

- **Idiomatic Q3 Design**: Mimics Q3's human input model (`usercmd_t` accumulation) rather than inventing a bot-specific scheme. This ensures bots can re-use the same downstream physics/combat pipeline as players.
- **No Semantic Arbitration**: `EA_*` functions never reject or prioritize conflicting commands—they just OR flags or overwrite scalar fields. All conflict resolution happens in the engine's move/combat code, preserving separation of concerns.
- **Late-Binding State**: Functions like `EA_Attack` don't immediately fire; they set `ACTION_ATTACK` for the engine to interpret. This allows the engine to consider current weapon, ammo, reload state, etc., before executing.
- **Contrast with Modern ECS**: Modern game engines (Unity, Unreal, custom ECS) often use event queues or command objects. Q3's flat struct with bitflags is more cache-friendly and faster for tight per-frame loops but less flexible for extension.
- **Stateful vs. Stateless**: The `botinputs` array is stateful (persists across frames); only `EA_ResetInput` clears it. This is opposite to pure event-based systems and requires careful phase ordering in the caller.

## Potential Issues

**None clearly inferable from code + context**, but worth noting:

- **Thread-unsafe by design**: If the server's frame loop and bot AI think calls ever ran in parallel (as mentioned in `sv_main.c` for some operations), concurrent calls to `EA_*` on different clients would race on global module state (though `botinputs[client]` array itself is safe). Current architecture is single-threaded per frame.
- **No validation of client range**: Functions like `EA_Jump(client)` assume `0 <= client < botlibglobals.maxclients`. Buggy callers could index out of bounds. This is mitigated by the fact that only the server VM calls these, and it knows its valid client range.
- **Incomplete commented-out code**: `EA_EndRegular` is fully commented out, and `EA_GetInput` has commented reset code. This suggests evolutionary refactoring where the split of get vs. reset happened mid-development. No correctness issue, but slightly confusing for maintainers.
