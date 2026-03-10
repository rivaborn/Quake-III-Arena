# code/game/be_ea.h — Enhanced Analysis

## Architectural Role

The Elementary Actions (EA) module is the **lowest-level command abstraction layer in the botlib AI stack**. It sits at the boundary between high-level bot decision-making (goal selection, movement, weapon choice, dialogue) and the engine's client input pipeline. Each frame, AI layers accumulate intent via discrete `EA_*` function calls; the EA layer buffers these commands and, at frame end, aggregates them into a unified `bot_input_t` structure that the server (`sv_bot.c`) feeds into the standard client input/simulation path. This design isolates botlib's AI logic from network timing and engine-specific input encoding.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/ai_dmq3.c`**, **`code/game/ai_dmnet.c`**: Call `EA_Move`, `EA_View`, `EA_Action` (attack, crouch, jump) during FSM node execution to drive bot locomotion and combat
- **`code/game/ai_move.c`**: Calls `EA_Move`, `EA_Jump`, `EA_DelayedJump`, `EA_Crouch` during reachability-based movement to follow planned routes
- **`code/game/ai_goal.c`**, **`code/game/ai_weap.c`**: Call `EA_SelectWeapon` during goal execution and weapon selection logic
- **`code/game/ai_chat.c`**: Calls `EA_Say`, `EA_SayTeam` for in-game bot chat responses
- **`code/server/sv_bot.c`**: Calls `EA_GetInput` to retrieve accumulated per-frame input; calls `EA_EndRegular` and `EA_ResetInput` to manage frame lifecycle

### Outgoing (what this file depends on)
- **`bot_input_t` type** (defined in `code/botlib/be_aas_def.h` or game headers): The output structure this module aggregates into
- **`vec3_t` type** (from `code/game/q_shared.h`): Used for direction and view-angle parameters
- **Per-client state arrays** (internal to `botlib/be_ea.c`): Maintains `usercmd_t` buffers and command queues allocated by `EA_Setup`
- **No external engine dependencies in this header**: The abstraction is pure; all syscall routing happens in `botlib/be_ea.c` implementation

## Design Patterns & Rationale

**Frame-Accumulation Pattern**: Unlike traditional game input pipelines that immediately serialize a single user input each frame, EA uses a **deferred-aggregation** model. AI layers call multiple `EA_*` functions during a frame (e.g., `EA_Move` + `EA_View` + `EA_Action`), which cache their effects. Only at `EA_EndRegular` does the layer finalize and snapshot the complete input state. This decouples:
- **AI decision velocity** from **network packet timing** (frames can be longer/shorter without stalling bot decisions)
- **Discrete action calls** from **continuous state representation** (simplifies AI code; EA handles encoding)

**Per-Client Indexing** (client ID as first parameter): Reflects the stateless, **process-function** architecture of Q3's era, where VMs avoid global state and OOP. All functions are pure (no `this` pointer); state lives in engine-allocated arrays indexed by ID. This design survives in the botlib API boundary.

**Minimal Boundary Coupling**: The header declares only the public EA API; implementation details (buffering, encoding) are hidden. Callers never touch the internal `usercmd_t` representation directly. This limits fragility if the internal storage format changes (e.g., if swing-batching or movement quantization evolves).

## Data Flow Through This File

1. **Ingress** (during a bot's think cycle):
   - Higher-level AI (movement, combat, chat) call individual `EA_*` functions (stateless calls)
   - Each call modifies the bot's **pending usercmd buffer** (internal to `botlib/be_ea.c`)
   - Example: `EA_Move(botClient, dir, speed)` → writes `forwardmove`/`sidemove`/`upmove`; `EA_View(botClient, angles)` → writes pitch/yaw

2. **Processing** (within-frame accumulation):
   - Multiple `EA_*` calls compose: `EA_Crouch + EA_MoveForward + EA_View + EA_Attack` all blend into one usercmd
   - `EA_Command` and chat functions queue text commands for later dispatch (separate from button/movement state)

3. **Egress** (end-of-frame finalization):
   - `EA_EndRegular(botClient, thinktime)` signals frame completion; may apply time-scaling or frame-rate smoothing
   - `EA_GetInput(botClient, thinktime, &input)` reads the finalized `bot_input_t` into the caller's buffer
   - `sv_bot.c` then routes this input through the standard client-simulation pipeline (as if it came from a human player's `usercmd_t`)

4. **Reset**:
   - `EA_ResetInput(botClient)` zeroes all state for the next frame

## Learning Notes

**Idiomatic to Q3 / Mid-2000s Game Engines**:
- No constructors/destructors; explicit `EA_Setup` / `EA_Shutdown` bracket the module lifetime (predates RAII in game engine practice)
- Imperative, functional style (each `EA_*` call is a side-effect-driven command, not a property setter)
- Stateful per-frame buffering rather than immediate dispatch; this was common for **frame-sync consistency** (all bot actions committed atomically once per server frame)

**Connection to Engine Architecture**:
- This layer exemplifies **layered abstraction boundaries** in Q3: the game VM never directly constructs network packets; instead it calls high-level functions that eventually flatten to a single `usercmd_t` per frame
- Mirrors the **engine's own input handling** (`code/client/cl_input.c`): both accumulate button state + movement vectors, then finalize once per frame
- The botlib module is **pluggable** via `GetBotLibAPI()`; EA is part of that contract, ensuring server-side game logic can drive bots identically to human players

**Potential Design Observation**:
- The EA layer's minimal surface area (no state queries, only commands + setup/teardown) makes it **easy to mock or replace** during testing or for alternate AI subsystems (though Q3's shipped with just one botlib)

## Potential Issues

- **No per-bot setup/teardown in public API**: `EA_Setup` / `EA_Shutdown` are global; individual per-bot initialization is opaque. If a developer forgets to call `EA_ResetInput` before each frame, stale input from the prior frame could bleed through (though `sv_bot.c` should enforce this discipline).
- **Type mismatch risk**: The header uses `vec3_t` and `bot_input_t` declared elsewhere; a mismatch in struct layout between header and `be_ea.c` could silently corrupt data. This is mitigated by the fact that `be_ea.c` is part of botlib (tightly integrated), not a separate DLL.
