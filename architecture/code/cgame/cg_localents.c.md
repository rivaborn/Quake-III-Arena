# code/cgame/cg_localents.c

## File Purpose
Manages a fixed-size pool of client-side "local entities" (smoke puffs, gibs, brass shells, explosions, score plums, etc.) that exist purely on the client and are never synchronized with the server. Every frame, it iterates all active local entities and submits renderer commands appropriate to each entity type.

## Core Responsibilities
- Maintain a pool of 512 `localEntity_t` slots via a doubly-linked active list and a singly-linked free list
- Allocate and free local entities, evicting the oldest active entity when the pool is exhausted
- Simulate fragment physics: trajectory evaluation, collision tracing, bounce/reflect, mark/sound generation, and ground-sinking
- Drive per-type visual update functions (fade, scale, fall, explosion, sprite explosion, score plum, kamikaze, etc.)
- Submit all live local entities to the renderer each frame via `trap_R_AddRefEntityToScene`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `localEntity_t` | struct (typedef) | Single pooled local entity: trajectory, color, radius, light, mark/bounce type, embedded `refEntity_t` |
| `leType_t` | enum | Discriminates update behavior: `LE_MARK`, `LE_EXPLOSION`, `LE_FRAGMENT`, `LE_MOVE_SCALE_FADE`, etc. |
| `leFlag_t` | enum | Bit flags: tumble, puff-no-scale, sound-played markers (kamikaze) |
| `leMarkType_t` | enum | Type of decal a fragment leaves on impact (`LEMT_NONE`, `LEMT_BURN`, `LEMT_BLOOD`) |
| `leBounceSoundType_t` | enum | Sound category on fragment bounce (`LEBS_NONE`, `LEBS_BLOOD`, `LEBS_BRASS`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cg_localEntities` | `localEntity_t[512]` | global | Fixed backing store for all local entity slots |
| `cg_activeLocalEntities` | `localEntity_t` | global | Sentinel head/tail of the doubly-linked active list |
| `cg_freeLocalEntities` | `localEntity_t *` | global | Head of the singly-linked free list |

## Key Functions / Methods

### CG_InitLocalEntities
- **Signature:** `void CG_InitLocalEntities(void)`
- **Purpose:** Zero the pool, initialize the circular active sentinel, and chain all slots into the free list.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets all three global pool structures.
- **Calls:** `memset`
- **Notes:** Called at startup and on tournament restart.

---

### CG_AllocLocalEntity
- **Signature:** `localEntity_t *CG_AllocLocalEntity(void)`
- **Purpose:** Pop from the free list (evicting the oldest active entity if necessary), zero the slot, and link it at the head of the active list.
- **Inputs:** None
- **Outputs/Return:** Pointer to a zeroed, active `localEntity_t`
- **Side effects:** May call `CG_FreeLocalEntity` on the tail (oldest) active entity; modifies `cg_freeLocalEntities` and `cg_activeLocalEntities` links.
- **Calls:** `CG_FreeLocalEntity`, `memset`
- **Notes:** Guaranteed to succeed; never returns NULL.

---

### CG_FreeLocalEntity
- **Signature:** `void CG_FreeLocalEntity(localEntity_t *le)`
- **Purpose:** Unlink from the active doubly-linked list and push onto the free singly-linked list.
- **Inputs:** `le` — must be currently active (`le->prev != NULL`)
- **Outputs/Return:** None
- **Side effects:** Modifies neighbor `prev`/`next` pointers and `cg_freeLocalEntities`.
- **Calls:** `CG_Error` on invalid state
- **Notes:** Does not zero the slot.

---

### CG_AddFragment
- **Signature:** `void CG_AddFragment(localEntity_t *le)`
- **Purpose:** Per-frame update for physics fragments (gibs, brass): traces movement, handles bouncing/stopping/sinking, spawns blood trails, leaves marks, plays bounce sounds, submits render entity.
- **Inputs:** `le` with `leType == LE_FRAGMENT`
- **Outputs/Return:** None
- **Side effects:** May call `CG_FreeLocalEntity`; calls `trap_R_AddRefEntityToScene`; may modify `le->pos.trType` to `TR_STATIONARY`; calls `CG_BloodTrail`, `CG_FragmentBounceMark`, `CG_FragmentBounceSound`, `CG_ReflectVelocity`.
- **Calls:** `BG_EvaluateTrajectory`, `CG_Trace`, `trap_CM_PointContents`, `trap_R_AddRefEntityToScene`, `AnglesToAxis`
- **Notes:** Stationary fragments sink 16 units into the ground over `SINK_TIME` ms before removal; uses an explicit lighting origin to preserve lighting while sinking.

---

### CG_ReflectVelocity
- **Signature:** `void CG_ReflectVelocity(localEntity_t *le, trace_t *trace)`
- **Purpose:** Compute reflected velocity off a collision plane, apply `bounceFactor`, and stop the entity if energy is low.
- **Inputs:** `le`, `trace` with valid plane normal and fraction
- **Side effects:** Modifies `le->pos.trDelta`, `le->pos.trBase`, `le->pos.trTime`, and potentially `le->pos.trType`.
- **Calls:** `BG_EvaluateTrajectoryDelta`, `DotProduct`, `VectorMA`, `VectorScale`, `VectorCopy`

---

### CG_AddLocalEntities
- **Signature:** `void CG_AddLocalEntities(void)`
- **Purpose:** Per-frame entry point — walks the active list backwards, frees expired entities, and dispatches to per-type update functions via a switch on `leType`.
- **Inputs:** None (reads `cg.time`, the active list)
- **Outputs/Return:** None
- **Side effects:** Frees expired entities; submits renderer commands for all surviving entities.
- **Calls:** `CG_FreeLocalEntity`, `CG_Error`, `CG_AddSpriteExplosion`, `CG_AddExplosion`, `CG_AddFragment`, `CG_AddMoveScaleFade`, `CG_AddFadeRGB`, `CG_AddFallScaleFade`, `CG_AddScaleFade`, `CG_AddScorePlum`, and MISSIONPACK variants.
- **Notes:** Iterates backwards so newly spawned entities (trails, marks) generated mid-frame are present this frame.

---

### CG_AddScorePlum
- **Signature:** `void CG_AddScorePlum(localEntity_t *le)`
- **Purpose:** Render a floating score number beside a player, color-coded by score magnitude, with a sinusoidal lateral drift and alpha fade.
- **Calls:** `trap_R_AddRefEntityToScene`, `VectorMA`, `CrossProduct`, `VectorNormalize`, `sin`

- **Notes on trivial helpers:** `CG_AddFadeRGB`, `CG_AddMoveScaleFade`, `CG_AddScaleFade`, `CG_AddFallScaleFade`, and `CG_AddExplosion` are straightforward: compute a normalized lifetime `c`, update `shaderRGBA`/radius, evaluate trajectory, cull if camera is inside the sprite, then call `trap_R_AddRefEntityToScene`. `CG_BloodTrail` spawns `LE_FALL_SCALE_FADE` smoke puffs along a gib's path at 150 ms intervals. `CG_FragmentBounceMark` and `CG_FragmentBounceSound` fire once per fragment (they clear their type after use).

## Control Flow Notes
`CG_InitLocalEntities` is called from cgame init and tournament restart. `CG_AddLocalEntities` is called once per rendered frame from `CG_DrawActiveFrame` (in `cg_view.c`), after entities and marks are processed. New local entities are spawned during event handling (`cg_effects.c`, `cg_weapons.c`, `cg_events.c`) by calling `CG_AllocLocalEntity`.

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`
  - `CG_SmokePuff`, `CG_ImpactMark` — `cg_effects.c`, `cg_marks.c`
  - `CG_Trace` — `cg_predict.c`
  - `CG_GibPlayer` — `cg_effects.c`
  - `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_CM_PointContents`, `trap_S_StartSound`, `trap_S_StartLocalSound` — engine syscall layer
  - `cg`, `cgs` — global state in `cg_main.c`
