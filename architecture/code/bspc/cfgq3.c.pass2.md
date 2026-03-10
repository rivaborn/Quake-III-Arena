# code/bspc/cfgq3.c — Enhanced Analysis

## Architectural Role

This configuration file is a **bridge between offline and runtime bot systems**, parameterizing the BSPC compiler's physics simulation to match the Quake III Arena engine's movement model. During map compilation, BSPC consumes these constants to compute reachability links and travel costs for the AAS (Area Awareness System), generating a binary `.aas` file that botlib loads at runtime. The config ensures that bot navigation decisions (jump heights, step climbing, water behavior) match what players can actually perform in-game.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC compiler** (`code/bspc/bspc.c`, `code/bspc/be_aas_bspc.c`) — parses this file during the AAS compilation phase via the script/precompiler subsystem (`code/bspc/l_precomp.c`, `code/botlib/l_script.c`)
- **Build pipeline** — typically invoked offline as `bspc.exe -bsp2aas mapname.bsp` before shipping a map

### Outgoing (what this file depends on / influences)
- **botlib at runtime** — the compiled `.aas` file (output of BSPC using these constants) is loaded by `code/botlib/be_aas_main.c` and used by `AAS_PredictClientMovement`, `AAS_Reachability_Jump`, `AAS_Reachability_Ladder`, etc.
- **Game engine physics** (`code/game/g_local.h`, `code/game/bg_public.h`) — the numeric values must match or diverge intentionally (e.g., `phys_gravity 800` mirrors `g_gravity 800`)
- **Presence type definitions** — the `PRESENCE_NORMAL` and `PRESENCE_CROUCH` macros (lines 28–30) must match enum values in `code/botlib/be_aas_def.h` for correct bbox indexing during AAS file loading

## Design Patterns & Rationale

**Configuration Externalization**: Rather than hardcoding physics into the BSPC binary, Quake III separates game-specific parameters into a `.c` file parsed as data. This allows different games or mod variants to tune navigation without recompiling the tool.

**Dual Physics Model**: The engine maintains one physics model (in `bg_pmove.c`, `bg_slidemove.c`), while BSPC maintains a simplified clone (`code/botlib/be_aas_move.c`) for reachability prediction. The config bridges these by providing shared constants—preventing bots from claiming unreachable areas but avoiding per-frame synchronization overhead.

**Presence Hull Parameterization**: Two bounding boxes rather than dynamic per-bot hulls. This is idiomatic to Q3's offline-first approach: compile once per map, reuse for all bot clients. Contrast with modern engines (Unreal, Unity) that compute navmesh connectivity at edit-time in the editor.

## Data Flow Through This File

1. **Parse** → BSPC precompiler reads bbox and settings blocks into memory structures (`aas_settings_t`)
2. **Simulate** → During reachability analysis (`AAS_ContinueInitReachability`), BSPC uses `phys_gravity`, `phys_maxjump`, `phys_maxstep` to predict if a bot can traverse between areas
3. **Encode** → Travel costs (`rs_*` values) are stored in reachability link metadata; travel time estimates are cached
4. **Load** → At runtime, `AAS_LoadAASFile` deserializes the compiled bounds, reach links, and cluster portals; `AAS_PresenceTypeBoundingBox` retrieves bbox dimensions from presence type index
5. **Query** → Pathfinding (`AAS_AreaRouteToGoalArea`, `AAS_PredictRoute`) uses cached costs to favor low-cost traversals (e.g., walk) over high-cost ones (e.g., rocket jump)

## Learning Notes

- **Separation of Concerns**: Engine physics (deterministic player movement) and bot physics (reachability estimation) are decoupled; bots never actually run `Pmove`, they just plan using AAS.
- **Idiomatic Q3**: Explicit numeric tuning file vs. modern data-driven editors where navmesh connectivity is auto-computed. This required human expertise to tune correctly.
- **Cost Heuristics**: The `rs_*` values (e.g., `rs_rocketjump 500` vs. `rs_startwalk 70`) encode "pain" scores—bot pathfinder prefers low-cost moves. This abstraction predates modern AI utility systems.
- **Compile-time Offline Tool**: BSPC is a **pure offline compiler**, never linked into the runtime engine. Its config is **not** runtime-accessible; cvars like `g_gravity` are separate. This is a clean architectural boundary (no cross-layer dependencies).

## Potential Issues

- **Silent Physics Drift**: If `bg_pmove.c` is modified (e.g., `PM_Accelerate` threshold changes) but cfgq3.c is not updated, bots will predict unreachable or suboptimal paths without warning. No build-time validation exists.
- **Presence Type Indexing**: The `PRESENCE_NORMAL = 2` and `PRESENCE_CROUCH = 4` constants must **exactly match** the enum in `code/botlib/be_aas_def.h` (line 28–30 here vs. that header's definition). A mismatch silently corrupts bbox lookups during AAS file load.
- **Hardcoded Numeric Magic**: Values like `phys_maxsteepness 0.7` and `rs_falldamage10 500` lack inline documentation; unclear if these are angles (radians/degrees?), costs (milliseconds?), or abstract units. Maintainers must reverse-engineer intent from code.
