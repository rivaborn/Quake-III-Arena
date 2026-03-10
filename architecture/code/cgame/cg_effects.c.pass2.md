# code/cgame/cg_effects.c — Enhanced Analysis

## Architectural Role
This file implements the **client-side visual effects pipeline** for the cgame VM. It sits downstream of the event system (`cg_event.c` → `CG_EventHandling`) and weapon system (`cg_weapons.c`), converting server-authoritative game events (impacts, deaths, teleports) and client-side weapon fire into purely cosmetic local entities. All effects are **non-authoritative and non-networked**; the server neither knows nor cares about them. The file acts as a factory layer that marshals effect parameters into a uniform `localEntity_t` representation, which is then processed and rendered by `cg_localents.c:CG_AddLocalEntities()` each frame.

## Key Cross-References

### Incoming (who depends on this file)
- **`cg_event.c`** fires `CG_EventHandling` which dispatches on server-sent `EV_*` event types, calling functions here (e.g., `EV_EXPLOSION` → `CG_MakeExplosion`, `EV_GIB_PLAYER` → `CG_GibPlayer`, `EV_KAMIKAZE` → `CG_KamikazeEffect`)
- **`cg_weapons.c`** calls `CG_MakeExplosion` for weapon-impact explosions, `CG_SmokePuff` for smoke trails, and `CG_Bleed` for blood spatters
- **`cg_localents.c:CG_AddLocalEntities()`** processes the `localEntity_t` pool populated by this file, interpolating position, modulating color, and culling by lifetime

### Outgoing (what this file depends on)
- **`cg_localents.c:CG_AllocLocalEntity()`** — allocates slots from the fixed-size `cg_localents[512]` pool; **returns `NULL` if full** (unhandled)
- **`cgs.media.*`** — preloaded shader and model handles (`waterBubbleShader`, `smokePuffRageProShader`, `rocketExplosionShader`, `lightningShader`, `kamikazeEffectModel`, `invulnerabilityImpactModel`, etc.) cached during `CG_RegisterMedia()` at level load
- **`cg.time`** — global client render time (milliseconds elapsed since level start); used for all temporal calculations
- **`cgs.glconfig.hardwareType`** — GPU capability query; RagePro fallback (line 145)
- **`trap_S_StartSound()`** — positional sound syscall for `CG_ObeliskPain`, `CG_InvulnerabilityImpact`, `CG_InvulnerabilityJuiced`
- **Math library** — `VectorCopy`, `VectorNormalize`, `AxisClear`, `RotateAroundDirection`, `AnglesToAxis` (shared with renderer)

## Design Patterns & Rationale

**Factory + Type Dispatch:** Each function allocates a `localEntity_t` and assigns a `leType` enum (`LE_EXPLOSION`, `LE_SPRITE_EXPLOSION`, `LE_FRAGMENT`, `LE_FADE_RGB`, etc.). The downstream processor (`cg_localents.c`) dispatches on `leType` to apply type-specific lifetime, physics, and fade logic. This **decouples effect creation from effect runtime behavior**.

**Hardware Fallback (line 145):** RagePro GPUs cannot alpha-fade shaders; the code detects this and swaps to a pre-authored opaque shader (`smokePuffRageProShader`). This is a **pragmatic mid-2000s compatibility hack** — modern engines use GLSL to conditionally compile shader variants.

**Temporal Randomization:** `CG_MakeExplosion` adds a random offset (0–63 ms) to `startTime` to desynchronize multiple simultaneous explosions. This prevents all explosions from pulsing in sync, making them feel more organic. The offset is **added on the client side only**, so different clients may see slightly different timings (acceptable for decoration).

**Time-Based Interpolation:** All effects use absolute `cg.time` (not frame counts) with linear trajectory types (`TR_LINEAR`, `TR_GRAVITY`). This makes effects **frame-rate independent** and ensures smooth motion at any FPS.

## Data Flow Through This File

1. **Ingress:** Server event sent to client (e.g., `EV_EXPLOSION` packed into snapshot entity message)
2. **Event Decode:** `cg_event.c:CG_EntityEvent()` unpacks event and fires the corresponding function (e.g., `CG_MakeExplosion(origin, dir, ...)`)
3. **Allocation:** Function calls `CG_AllocLocalEntity()` to grab a slot from the 512-slot `cg_localents[]` pool
4. **Initialization:** Set `leType`, `startTime`/`endTime`, trajectory (`TR_LINEAR` / `TR_GRAVITY` with origin/velocity), render state (shader, model, color, rotation), and flags
5. **Queuing:** Local entity is now live in the pool; no explicit return-to-queue (it auto-expires when `cg.time > endTime`)
6. **Processing (per frame):** `cg_localents.c:CG_AddLocalEntities()` iterates the pool, updates position via trajectory math, modulates color by age, culls expired entities, and calls `trap_R_AddRefEntityToScene()` to render

## Learning Notes

- **Idiomatic to Q3A era:** Effects are hand-authored for specific events (not generated via particle systems). Each effect is a carefully tuned sprite or model with precalculated shaders. Modern engines use GPU particle systems or compute shaders.
- **Client-only optimization:** Because effects are non-authoritative, they can be arbitrarily complex (many simultaneous gibs, explosions) without affecting server CPU. This is a **latency win** — no network sync overhead.
- **Deterministic randomness:** Uses `rand()` and `crandom()` (seeded at engine startup) rather than true randomness. This means all clients see identical random effects if they seed identically — useful for demo playback fidelity (though not required here).
- **Shared asset precaching:** All `cgs.media.*` references are **preloaded during `CG_RegisterMedia()` at level load**. This avoids per-effect alloc overhead and ensures deterministic memory layout.
- **No error recovery:** If `CG_AllocLocalEntity()` returns `NULL` (pool full), callers don't check. In practice, the 512 slots rarely saturate, but a defensive game would fall back gracefully.

## Potential Issues

- **Pool exhaustion:** `CG_AllocLocalEntity()` can return `NULL` (code never checked here). With 512 gibs/particles + user effects, the pool can theoretically fill. Modern fix: resize pool or use ring buffer eviction.
- **Static state fragility:** `lastPos` in `CG_ScorePlum` and `seed` in `CG_SmokePuff` are function-local statics. In splitscreen or netplay replay contexts, this can cause subtle cross-client desync. Ideally, these would be keyed by client ID or encapsulated in a struct.
- **No bounds on trajectory velocity:** Gibs use `crandom()*300` for velocity but no max clamp. Extreme randomness could cause gibs to visually escape the level. Not a bug, but potentially surprising.
- **MISSIONPACK ifdef fragmentation:** Functions like `CG_KamikazeEffect`, `CG_LightningBoltBeam`, `CG_InvulnerabilityImpact` are only compiled for MissionPack builds. The baseline client has no fallback, so events for these effects would be silently ignored (acceptable but could confuse mod authors).
