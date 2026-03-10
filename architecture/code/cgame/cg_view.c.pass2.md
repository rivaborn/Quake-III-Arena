# code/cgame/cg_view.c — Enhanced Analysis

## Architectural Role

This file is the **central per-frame orchestration hub** for cgame's visual output pipeline. It sits at the boundary between **server-authoritative snapshot delivery** and **client-side visual synthesis**, driving prediction, view setup, scene population, and final render submission. Every rendered frame flows through `CG_DrawActiveFrame`—called once per frame by `code/client/cl_cgame.c` after network snapshot arrival and before renderer execution. It is the top of cgame's render call graph and orchestrates handoffs to five subsystems: prediction (divergence decay), entity rendering, audio spatialization, HUD composition, and renderer queuing.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/client/cl_cgame.c`** — calls `CG_DrawActiveFrame` once per rendered frame (entry point into cgame frame logic)
- **Console input** — `testmodel`/`testgun`/`nextframe`/`nextskin` developer commands (bound in `q3default.cfg`)

### Outgoing (what this file depends on)
- **Prediction:** `CG_PredictPlayerState`, `CG_Trace`, `CG_PointContents` from `cg_predict.c`
- **Entity rendering:** `CG_AddPacketEntities`, `CG_AddLocalEntities`, `CG_AddMarks`, `CG_AddParticles` from `cg_ents.c`, `cg_localents.c`, etc.
- **Weapon/HUD:** `CG_AddViewWeapon` from `cg_weapons.c`; HUD/2D rendering deferred to `CG_DrawActive` in `cg_draw.c`
- **Audio:** `trap_S_Respatialize` (listener position/orientation), `CG_PlayBufferedSounds`, `CG_PlayBufferedVoiceChats`, `CG_PowerupTimerSounds`
- **Renderer:** `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `CG_DrawActive`
- **Math utilities:** `AnglesToAxis`, `AngleVectors`, `VectorMA`, `DotProduct` from `q_math.c`
- **Global state:** `cg`, `cgs`, cvars (`cg_fov`, `cg_viewsize`, `cg_gun_x/y/z`, `cg_thirdPersonRange`, etc.)

## Design Patterns & Rationale

### 1. **Prediction-First Architecture**
Call sequence: snapshot → prediction → refdef build → entity add. This pattern **decouples visual presentation from network latency**; unacknowledged client commands run locally via `CG_PredictPlayerState`, allowing smooth first-person motion even with packet loss. The refdef is built *after* prediction completes, so view offset calculations use predicted viewheight/velocity.

### 2. **Compositional View Perturbation**
First-person view (CG_OffsetFirstPersonView) **composes multiple independent offset contributions** in sequence:
- Weapon kick (angles)
- Damage kick (decaying angular transients)
- Velocity-based pitch/roll (head-relative aiming feedback)
- Head bob (vertical + rotational, clamped while crouching)
- Duck/step/land smoothing (position deltas fading over frame windows, ~200ms each)

Each contributor is time-bounded; composing them allows designers to tune each independently without nonlinear side effects.

### 3. **Circular Ring Buffer for Sound**
`CG_AddBufferedSound` enqueues sounds into `cg.soundBuffer[CG_MAX_ANNOUNCE_SOUNDS]` with wrap-around index (`cg.soundBufferIn`). This **serializes announcer/powerup-expiry sounds** to prevent acoustic clipping; `CG_PlayBufferedSounds` drains the queue. Trade-off: fixed capacity vs. simplicity (no dynamic allocation in tight loop).

### 4. **Third-Person Camera with Collision**
`CG_OffsetThirdPersonView` traces a small AABB (4×4×4) from player eye to desired camera position, **preventing clipping through walls** while maintaining focus on the player. The trace-and-retry pattern (attempt once, if clipped, pop upward and retry) handles **tunnel ceiling edge cases**.

### 5. **Embedded Developer Infrastructure**
Model testing functions (`CG_TestModel_f`, `CG_TestGun_f`, frame/skin cycling) are **compiled into release code**, not behind `#ifdef DEBUG`. This reflects Q3A's ethos of shipping accessible modding/debugging tools; no runtime overhead unless commands are invoked.

## Data Flow Through This File

```
Server Snapshot
    ↓
CG_DrawActiveFrame (entry)
    ├→ CG_ProcessSnapshots (advance snapshot ring buffer)
    ├→ CG_PredictPlayerState (run unacked commands locally)
    ├→ CG_CalcViewValues (master refdef setup)
    │  ├→ CG_CalcVrect (viewport from cg_viewsize)
    │  ├→ CG_OffsetFirstPersonView or CG_OffsetThirdPersonView (view offset)
    │  ├→ CG_CalcFov (FOV + zoom interpolation + water warp)
    │  └→ Convert angles to axis matrix
    ├→ CG_DamageBlendBlob (blood decal overlay)
    ├→ CG_AddPacketEntities (all server entities)
    ├→ CG_AddMarks / CG_AddParticles / CG_AddLocalEntities (client-side)
    ├→ CG_AddViewWeapon (first-person weapon model)
    ├→ CG_AddTestModel (optional debug model)
    ├→ CG_PlayBufferedSounds (drain announcer queue)
    ├→ trap_S_Respatialize (listener origin + axis → audio engine)
    └→ CG_DrawActive (2D HUD + final render submission)

Output: refdef_t queued in renderer, scene submitted, audio spatialized
```

## Learning Notes

### Idiomatic to Quake III Arena / Early-2000s Engine Design
- **Cvar-driven cosmetics:** Player-facing parameters (gun position offsets, bob amplitude, camera distance) exposed as cvars for live tuning; no recompile needed
- **Frame-delta-based smoothing:** Duck/step/land offsets use frame time deltas (`cg.time - cg.duckTime`) to interpolate over fixed windows (DUCK_TIME, STEP_TIME). Robust across variable frame rates
- **Early-exit patterns:** Multiple returns for special cases (intermission, loading screen, dead player) reduce nesting; code reads top-to-bottom
- **Hardware-specific fallbacks:** `CG_DamageBlendBlob` skips on GLHW_RAGEPRO (old Intel GMA); reflects era of wide GPU capability variance
- **Inline math:** No matrix/quaternion library calls; angle/vector math done inline with `AngleVectors`, `VectorMA`, `DotProduct` for inlining by compiler

### Modern Engines Do This Differently
- **ECS systems:** Modern engines separate view transform (camera system) from entity rendering (spatial query + draw). Q3A conflates them in one orchestrator
- **Prediction reconciliation:** Modern netcode uses **full snapshot replay with client input replay** (lag compensation); Q3A uses simpler **delta decay** (slow divergence correction)
- **SIMD matrix math:** Q3A uses scalar angle/vector ops; modern engines batch view/projection matrices through SIMD
- **Asset streaming:** cgame is responsible for all asset precaching; modern engines defer to asset managers

## Potential Issues

1. **Unbounded test model re-registration:** `CG_AddTestModel` calls `trap_R_RegisterModel` every frame without checking if the model is already cached. While the renderer likely caches internally, this is inefficient for developer tools.

2. **Damage blend blob skipped on old hardware:** `CG_DamageBlendBlob` is conditionally compiled/skipped on GLHW_RAGEPRO, creating visual inconsistency for users on legacy GPUs. Modern fallback would be a 2D screen-space fade.

3. **No overflow protection on sound buffer:** `CG_AddBufferedSound` wraps `soundBufferIn` with modulo but assumes the queue won't overflow if announcer sounds are spaced >5 frames apart. If map creates many simultaneous events (powerup expire + kill announce + voicechat), older samples could be lost silently.

4. **Trace-only collision for third-person camera:** `CG_OffsetThirdPersonView` uses collision traces but not dynamic entity collision. If a large entity (e.g., a moving platform) is between player and camera, it may clip through. Modern engines would query dynamic AABBs.
