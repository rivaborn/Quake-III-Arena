# code/game/be_aas.h — Enhanced Analysis

## Architectural Role

This file defines the **public data-type interface** between the game VM's bot AI subsystem and the botlib AAS library. It serves as the game-side half of a syscall-based VM→engine boundary. Game AI code (`ai_dmq3.c`, `ai_dmnet.c`, `g_bot.c`) never calls botlib functions directly; instead, it invokes `trap_BotLib*` syscall wrappers that marshal these types to the server, which then dispatches to the actual botlib implementation. This isolation enables runtime VM polymorphism (QVM, native DLL, interpreted) and security sandboxing while keeping botlib completely self-contained.

## Key Cross-References

### Incoming (Game VM Layer)
- **code/game/ai_dmq3.c, ai_dmnet.c, ai_team.c**: Call `trap_BotLibMovement*`, `trap_BotLibRouting*` functions using these types; consume `aas_entityinfo_t`, `aas_clientmove_t`, `aas_predictroute_t`
- **code/game/g_bot.c**: Bot lifecycle management; uses `aas_entityinfo_t` for entity state snapshots
- **code/game/be_ea.h**: Imported alongside this header; defines `bot_input_t` accumulation layer

### Outgoing (botlib / Server / Common)
- **code/botlib/be_aas_*.c**: All actual implementation (be_aas_route.c, be_aas_move.c, be_aas_entity.c, etc.) defines functions whose return types are declared here
- **code/server/sv_game.c**: `SV_GameSystemCalls` dispatcher routes game VM syscalls (opcode range 200–599) to botlib vtable functions
- **code/qcommon/q_shared.h**: Provides `qboolean`, `vec3_t`, `cplane_t` base types
- **code/botlib/be_interface.c**: Exposes `botlib_export_t` vtable to server; all functions here are type-checked against this header

## Design Patterns & Rationale

**Syscall-based VM Boundary:**
Rather than linking botlib directly into the game VM, Quake III uses numbered syscall opcodes (200–599) to invoke botlib services. This pattern:
- Allows the game VM to execute as QVM bytecode, native x86/PPC, or via interpreter — all transparently
- Prevents malicious/buggy bot code from directly corrupting engine memory
- Enables runtime replacement of the AI module without VM recompilation

**Travel Type Flags (TFL_*):**
A 32-bit bitfield enumerates movement capabilities. This is more flexible than a single movement mode enum:
- Bitmask composition allows queries like "find routes using walking OR crouching, but not rocket jumps"
- Reflects Quake III's rich movement vocabulary: normal walking, rocket/BFG/strafe jumping, grappling, teleportation, func_bob riding, etc.
- The `TFL_DEFAULT` macro defines the baseline moveset for standard bot pathfinding

**Stop-Event Flags (SE_* and RSE_*):**
Rather than returning a boolean "hit something," movement and route prediction return an **event code** explaining *why* prediction halted:
- `SE_HITGROUND`, `SE_LEAVEGROUND`, `SE_ENTERWATER` provide rich feedback for bot decision-making
- `RSE_NOROUTE`, `RSE_USETRAVELTYPE` signal specific routing failures
- This allows bot code to adapt strategy based on what obstacle was encountered

**Result Structures:**
`aas_trace_t`, `aas_clientmove_t`, `aas_predictroute_t` return **compound results** (position + area + event + metadata) rather than pointer chains. This matches the engine's preference for flat, copyable structures suitable for network transmission and VM marshaling.

## Data Flow Through This File

1. **Bot AI Decision Loop** (game VM):
   - Calls `trap_BotLibRouting(...)` to find next waypoint
   - Passes goal area, current area, travel-type filter (TFL_*), and optional alternate-goal hints

2. **Syscall Marshal** (server/sv_game.c):
   - Server receives syscall opcode + args in message buffer
   - Converts game-VM types → botlib types (minimal translation needed)

3. **Routing Execution** (botlib/be_aas_route.c):
   - Executes `AAS_AreaRouteToGoalArea(...)`, which returns `aas_predictroute_t`
   - Struct contains: endpoint area, stop-event reason, path segments, travel flags used

4. **Return Path**:
   - Result marshaled back to game VM in same struct form
   - Bot code unpacks `aas_predictroute_t.endarea`, checks `aas_predictroute_t.stopevent`
   - Feeds into FSM state machine (e.g., `AINode_DeathmatchMove` in ai_dmnet.c)

## Learning Notes

**VM Boundary Design:**
This is a textbook example of **secure VM isolation via syscall abstraction**. Compare to modern game engines:
- Unreal/Unity link scripting DLLs directly → VM crashes can corrupt engine
- Quake III marshals all cross-boundary calls through numbered opcodes → VM is sandboxed

This design choice made sense in 2000 when VMs were less trusted and the codebase needed to run on multiple CPU architectures.

**Travel Type Encoding:**
The 20+ `TFL_*` flags reflect deep thought about movement modeling. Modern engines often use a single "movement mode" enum; Quake III's bitfield approach is more expressive for navmesh queries and bot skill variation ("this bot uses rocket jumps, that one doesn't").

**Stop Events:**
The granularity of `SE_*` flags (gap detection, damage detection, cluster portal crossing) reveals that botlib was designed for **continuous monitoring during movement prediction**, not just start/end points. This enables bots to react dynamically (e.g., "avoid paths through lava").

## Potential Issues

**Minor:** The `MAX_STRINGFIELD` guard (lines 33–35) is defensive but not strictly necessary — this constant is typically defined by botlib headers, and re-guarding it here is harmless.

**Historical:** The commented-out `bsp_trace_t` block (lines 98–117) references a type defined in `botlib.h` but excluded via comment to avoid duplication. This is valid but leaves dead code; could be cleaned up in a refactoring pass.
