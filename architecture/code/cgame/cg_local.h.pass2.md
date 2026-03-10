# code/cgame/cg_local.h — Enhanced Analysis

## Architectural Role

This header is the **internal contract binding all cgame VM subsystems together**. As a QVM-hosted client-side game module, cgame is completely isolated from the engine's renderer, server, and other subsystems—it communicates exclusively through ~80 `trap_*` syscalls defined here. This file centralizes the volatile and static state (`cg_t` and `cgs_t`), the entity representation (`centity_t`), local transient effects (`localEntity_t`), and all media caches (`weaponInfo_t`, `clientInfo_t`) needed to consume snapshots, predict player movement, interpolate entities, and drive the 3D scene generation.

## Key Cross-References

### Incoming (who depends on this file)
- **All cgame/*.c files** unconditionally include `cg_local.h` to access `cg`, `cgs`, `cg_entities[]`, global externs, and function prototypes.
- **Client engine** (`code/client/cl_cgame.c`) loads the cgame VM and dispatches to `CG_DrawActiveFrame` export, but only sees `cg_public.h`—it does **not** see this private header.
- **Renderer** receives refEntity_t and poly_t from cgame but has no knowledge of `centity_t`, `lerpFrame_t`, or the internal cgame state machine.

### Outgoing (what this file depends on)
- **Includes:** `q_shared.h` (entity state, player state, types), `bg_public.h` (gameplay constants, animation, items), `tr_types.h` (renderer types: `refEntity_t`, `refdef_t`, `polyVert_t`), `cg_public.h` (snapshot, import/export API).
- **Trap syscalls:** All cgame→engine communication is **exclusively through trap stubs** (`trap_R_*` for renderer, `trap_S_*` for sound, `trap_CM_*` for collision, `trap_Snapshot*` for network state, `trap_Cvar_*` for console variables, etc.). The cgame module is a complete sandbox with zero direct C linkage to engine internals.
- **Shared game code:** `bg_pmove.c`, `bg_misc.c`, `q_math.c` must be compiled identically into both cgame and game VMs to ensure deterministic client-side prediction.

## Design Patterns & Rationale

**1. Double-Buffered Snapshot Pipeline**  
`snap` and `nextSnap` point to active and future snapshots. Interpolation fraction (`frameInterpolation`) blends between them. Rationale: Server transmits snapshots at ~10 Hz; client renders at 60+ Hz; this pattern decouples network ticks from render frames and enables smooth motion.

**2. Per-Entity Interpolation State (lerpFrame_t)**  
Each player has separate `legs`, `torso`, `flag` lerpFrame structs tracking animation progression, frame time, yaw/pitch. Rationale: Quake's model system uses separate torso/legs bones; interpolation must track each independently to blend animations smoothly.

**3. Entity Error Correction (errorOrigin, errorAngles, extrapolated)**  
When server position/angle jumps (teleport, big lag correction), cgame stores the error and decays it over `errorTime`. Rationale: Sudden position snaps are visually jarring; smooth decay toward authoritative state looks better.

**4. Client-Side Prediction as Parallel State**  
`predictedPlayerState` and `predictedPlayerEntity` are **separate** from snapshot entities. They replay user commands through `Pmove` to predict position before server ack. Rationale: Network latency (~100ms) creates perceived input lag; local prediction masks it. Divergence from server is corrected by error decay.

**5. vmCvar_t Batch Registration**  
~80 cvar externs all point to CVARs registered once at `CG_Init`. Rationale: Defers all cvar lookups to initialization; per-frame access is just a read of the cached value, not a string hashtable lookup.

**6. Fixed-Size Pools with Linked Lists**  
`localEntity_t` (256 max), `markPoly_t` (256 max), entity array (1024). Rationale: Q3 pre-allocates everything in level memory; no malloc/free per-frame. Linked lists allow efficient iteration and removal.

**7. Dynamic Function Pointers in Media Structs**  
`weaponInfo_t.missileTrailFunc`, `ejectBrassFunc` are function pointers dispatching to weapon-specific visual logic (railgun vs. rocket vs. shotgun trails). Rationale: Avoids giant per-weapon switch statements in rendering code; behavior is data-driven.

## Data Flow Through This File

1. **Snapshot Ingestion** (`CG_ProcessSnapshots`)
   - Engine delivers new snapshot → `cg.snap` and `cg.nextSnap` pointers advance
   - Server commands processed → may trigger team/scoreboard/configstring updates
   
2. **Entity Snapshot → Centity_t**
   - Each `entityState_t` in snapshot finds corresponding `cg_entities[entityNum]` slot
   - `currentState` updated; `interpolate` flag set if `nextState` also valid
   
3. **Prediction** (`CG_PredictPlayerState`)
   - Replays unacknowledged user commands through `Pmove`
   - Stores result in `predictedPlayerState`; server response eventually syncs it
   
4. **Per-Frame Rendering** (`CG_DrawActiveFrame`)
   - Calls `CG_AddPacketEntities` → iterates snapshot entities, dispatches per-type rendering (player, item, missile, mover, beam, etc.)
   - Calls `CG_AddLocalEntities` → updates and renders all local effects
   - Calls `CG_AddMarks` → renders all decal polygons
   - Submits refEntity_t and poly_t to renderer via `trap_R_AddRefEntityToScene`
   
5. **Event Firing**
   - `CG_CheckEvents` translates `EV_*` entity events into audio cues, visual spawns, screen shakes
   - May spawn new local entities (explosions, impacts, scoring plums)

6. **HUD Rendering** (`CG_Draw2D`)
   - Draws status bar, crosshair, chat, scores, team overlays using 2D renderer calls
   - Uses draw time (`cg.time`) for animation and oscillation

## Learning Notes

**VM Architecture Pattern**
- Q3's VM design is a microcosm of OS kernel/userspace: strict syscall ABI boundary, sandboxed code execution, no direct memory access to engine.
- This is very different from modern engines (Unreal, Unity) where game code is native and has direct access to all engine internals.
- The benefit is **mod-safety** and **cross-platform bytecode portability**; the cost is **syscall overhead** and **inability to do hand-optimized SIMD**.

**Interpolation as a Core Primitive**
- The entire `lerpFrame_t` system would be unfamiliar to ECS or modern engines, which typically use linear interpolation on component positions.
- Q3 pre-computes per-frame animation frames offline; runtime just blends between them. Modern engines compute skeletal poses at runtime.
- This reflects Q3's era (2000s): CPU was constrained, GPU was emerging, so pre-baked animation frames were efficient.

**Prediction Divergence**
- The separation of `predictedPlayerState` (client) vs. `snap->ps` (server) is fundamental to latency compensation.
- A "bad predict" (e.g., predicted you jumped when server said you didn't) causes visible correction jumps. Modern engines use rollback/resimulation to avoid this.

**State Lifetime Surprise**
- The comment "cgame module is unloaded and reloaded on each level change; NO persistent data between levels" is critical.
- This means every global in this file is **ephemeral**. You cannot store player data in `cg` across map changes. That's why all persistent data (scores, player stats) lives in the server or in cvars.

**Snapshot Culling via PVS**
- The renderer does PVS frustum culling; cgame doesn't. cgame just renders everything in `cg.snap`, trusting the server's PVS-based entity list.
- This is a clean separation: server controls **what entities exist**; renderer controls **what's visible in view frustum**.

## Potential Issues

1. **Unbounded Prediction Divergence**
   - If network latency is extreme or packet loss is high, `predictedPlayerState` may diverge far from `snap->ps`. The error decay over `errorTime` helps, but on very bad networks the correction jump is still noticeable.
   - No fallback to "pause prediction" on extreme divergence.

2. **Fixed-Size Entity Array**
   - `cg_entities[MAX_GENTITIES]` is 1024 (hardcoded). On a hugely populated map (800+ players in a future mod), this overflows. No dynamic reallocation.

3. **Animation Frame Corruption**
   - `lerpFrame_t.animation` is a raw pointer to `clientInfo_t.animations[]`. If a client's model changes mid-frame, this pointer becomes dangling. No validation between frames.

4. **Local Entity Pool Fragmentation**
   - `localEntity_t` is a fixed 256-entry pool (inferred from `MAX_MARK_POLYS`). Long play sessions with many effect spawns and early deaths could fragment the pool, causing "no more local entities" failures silently.

5. **Trajectory Integration Precision**
   - `localEntity_t.pos` and `.angles` use `trajectory_t` (likely simple Euler integration). For long-lived particles or projectiles, accumulated error could cause visible drift or jitter.

6. **No Partial Snapshot Handling**
   - If a snapshot arrives incomplete (fragmented packet loss), cgame assumes `snap` is always valid before using it. No null checks visible in the typedef; relies on caller discipline.

---

## Summary

`cg_local.h` is the **architectural nucleus of cgame's data and control model**. Its key role is:
- **Isolation boundary:** Trap syscalls enforce a clean VM sandbox.
- **State machine:** Double-buffered snapshots, prediction state, local transients, and per-frame time all coordinated here.
- **Type vocabulary:** Every cgame subsystem speaks in terms of `centity_t`, `localEntity_t`, `lerpFrame_t`, and the two globals `cg`/`cgs`.
- **Era artifact:** The hardcoded pools, animation frame pre-baking, and interpolation-centric design reflect 2000s-era constraints and practices that differ fundamentally from modern engines.
