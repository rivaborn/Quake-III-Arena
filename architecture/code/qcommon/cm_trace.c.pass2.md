# code/qcommon/cm_trace.c — Enhanced Analysis

## Architectural Role

`cm_trace.c` is the collision kernel of the entire engine — the single file that every moving thing (players, projectiles, entities, bots) passes through on every frame. It sits inside `qcommon`, making it shared infrastructure consumed by both the authoritative server and the client-side prediction path. The server calls it through `SV_Trace` → `CM_TransformedBoxTrace` for all entity movement and damage traces; the cgame VM calls it through `trap_CM_BoxTrace` syscalls for client-side `Pmove` prediction, which must produce byte-identical results to the server to avoid visible misprediction divergence. The botlib also wraps it via `AAS_Trace` in `be_aas_bspq3.c` for movement reachability validation, using the `botlib_import_t` vtable. All three consumers share the same `clipMap_t` global (`cm`) loaded once by `cm_load.c`.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_world.c`** — `SV_Trace` / `SV_PointContents` are the primary callers of `CM_BoxTrace` and `CM_TransformedBoxTrace` each server frame. Every `G_MissileImpact`, player ground-check, and entity link query routes through here.
- **`code/game/g_active.c`, `g_weapon.c`, `g_missile.c`** — indirect via `trap_Trace` (opcode dispatched by `sv_game.c`'s `SV_GameSystemCalls`) which calls `CM_BoxTrace`.
- **`code/cgame/cg_predict.c`** — `CG_PredictPlayerState` calls `trap_CM_BoxTrace` / `trap_CM_TransformedBoxTrace` via the `CL_CgameSystemCalls` bridge; must be the same code path as the server or prediction artifacts appear.
- **`code/botlib/be_aas_bspq3.c`** — `AAS_Trace` calls `CM_BoxTrace` through `botimport.trace`; used for reachability link validation and entity collision.
- **`code/botlib/be_aas_move.c`** — `AAS_ClipToBBox`, `AAS_TraceClientBBox` ultimately reach `CM_BoxTrace` through the botlib import table.
- **`c_traces`, `c_brush_traces`, `c_patch_traces`** — read by `common.c` / console for `cm_trace_stats` display.

### Outgoing (what this file depends on)

- **`code/qcommon/cm_patch.c`** — `CM_TraceThroughPatchCollide`, `CM_PositionTestInPatchCollide`; Bézier surface intersection not implemented here.
- **`code/qcommon/cm_load.c` / `cm_test.c`** — `CM_BoxLeafnums_r`, `CM_StoreLeafs`, `CM_ClipHandleToModel`, `CM_ModelBounds`, `CM_TempBoxModel`; BSP leaf enumeration and model-handle resolution.
- **`cm` global (`clipMap_t`)** — populated by `cm_load.c`; this file is purely a consumer. All node, leaf, brush, surface arrays are read but never written except `checkcount` fields.
- **`cm_noCurves` cvar** — read each leaf traversal to optionally skip patch collision; registered in `cm_load.c` / `cm_main.c`.
- **`q_shared.h` math** — `DotProduct`, `VectorMA`, `VectorNormalize`, `AngleVectors`, `Square`, `VectorLengthSquared`; no renderer or sound dependencies.

## Design Patterns & Rationale

**Slab method (Cyrus–Beck / Liang–Barsky) for convex brushes.** `CM_TraceThroughBrush` maintains `enterFrac`/`leaveFrac` across all half-spaces; if `enterFrac > leaveFrac` the ray misses. This is optimal for convex polyhedra and is standard in all BSP-era engines. The tradeoff is it only works for convex shapes — hence why patches require a completely separate `cm_patch.c` pipeline.

**Capsule = cylinder body + two sphere caps.** Rather than representing the player as a pure AABB (which causes corner-catching on edges), the capsule code decomposes collision into `CM_TraceThroughVerticalCylinder` plus two `CM_TraceThroughSphere` calls. The `sphere_t` added to `traceWork_t` carries only the extra offset and radius, keeping the AABB path zero-cost for non-capsule traces.

**Role-swap for AABB-vs-capsule.** `CM_TestBoundingBoxInCapsule` / `CM_TraceBoundingBoxThroughCapsule` swap the roles of the two volumes — converting the AABB into a temporary capsule and the capsule model into a temporary box (`CM_TempBoxModel`). This lets the same underlying `CM_TestInLeaf` / brush-trace code handle both directions without a separate implementation.

**`checkcount` deduplication.** Brushes and patches each carry a `checkcount` field tested against `cm.checkcount` (incremented per-trace). This prevents double-testing objects that appear in multiple BSP leaves — a correctness requirement for the BSP multi-leaf overlap case with zero runtime overhead beyond a single integer comparison.

**`SquareRootFloat` (fast inverse rsqrt).** The 0x5f3759df magic constant (popularized as "Carmack's constant", though attributed to Greg Walsh) gives ~1% error after two Newton–Raphson iterations — sufficient for collision epsilon tolerances where the exact value is only used to find intersection fractions clamped to [0,1].

## Data Flow Through This File

```
External callers
  SV_Trace / trap_CM_BoxTrace / AAS_Trace
        │
        ▼
  CM_TransformedBoxTrace     ← transforms start/end to model-local space,
  CM_BoxTrace                   rotates result plane back to world
        │
        ▼
  CM_Trace                   ← builds traceWork_t; dispatches:
  ├─ position test? ──────────► CM_PositionTest → CM_BoxLeafnums_r → CM_TestInLeaf
  │                                                                    ├─ CM_TestBoxInBrush
  │                                                                    └─ CM_PositionTestInPatchCollide
  ├─ capsule-vs-capsule? ─────► CM_TraceCapsuleThroughCapsule
  │                              ├─ CM_TraceThroughVerticalCylinder
  │                              └─ CM_TraceThroughSphere (×2)
  ├─ bbox-vs-capsule model? ──► CM_TraceBoundingBoxThroughCapsule
  └─ world sweep? ────────────► CM_TraceThroughTree (recursive BSP descent)
                                  └─ CM_TraceThroughLeaf
                                      ├─ CM_TraceThroughBrush  (slab method)
                                      └─ CM_TraceThroughPatch → cm_patch.c
        │
        ▼
  trace_t *results            ← fraction, plane, surfaceFlags, contents, endpos
```

The `traceWork_t` is allocated on the stack in `CM_Trace` and passed by pointer everywhere — no heap allocation occurs during a trace. Contact plane is stored as a pointer into the permanent `cm.planes` array, not copied, so callers see the actual BSP plane data.

## Learning Notes

- **This is the "inner loop" of a game engine.** In a 64-player server frame, `CM_TraceThroughBrush` may execute thousands of times. Understanding the slab method here is foundational to all subsequent collision work — Unity's CharacterController, Unreal's `SweepSingleByChannel`, and Godot's `move_and_slide` all descend from this conceptual lineage.
- **BSP as a spatial index vs. scene graph.** Modern engines use BVH (bounding volume hierarchies) for dynamic scenes; Q3's BSP serves as both the static world collision structure and the visibility culling structure. `CM_TraceThroughTree` exploits the BSP's spatial ordering to cull entire subtrees via the plane/offset test — the same geometry that the renderer uses for PVS.
- **Shared physics code (`bg_pmove.c`) depends on this being deterministic.** The `trap_CM_*` syscall mechanism means the cgame VM and server VM call the same native function with the same `cm` state. Any floating-point non-determinism here would break prediction. The `SURFACE_CLIP_EPSILON` and the conservative `-1/+1` leaf bound expansion in `CM_PositionTest` are deliberate robustness choices, not accidents.
- **Capsule vs AABB is still approximate for rotated sub-models.** `CM_TransformedBoxTrace` comments acknowledge that rotating an AABB is only approximate; only the capsule path transforms exactly. This is a real tradeoff that modern engines resolve with OBB or convex hull support.
- **The `// bk: FIXME` annotations on `RotatePoint`/`TransposeMatrix`** show const-correctness issues with the 3×3 matrix type (`vec3_t[3]`) — a common pain point in C97 codebases where array-of-array types cannot be const-qualified through multiple pointer levels.

## Potential Issues

- **`MAX_POSITION_LEAFS = 1024` is a hard cap.** If `CM_BoxLeafnums_r` overflows (large AABB touching more than 1024 leaves, e.g. a teleporter brush spanning many sectors), `ll.overflowed` is set but the existing partial leaf list is still tested. The trace silently succeeds with incomplete data rather than failing safe.
- **`SquareRootFloat` with zero input.** The fast-rsqrt path is undefined for `number == 0` (produces NaN/Inf depending on the IEEE implementation of the integer tricks). Callers guard this via distance checks before calling, but the function itself has no assertion.
- **`CM_TestBoundingBoxInCapsule` mutates `tw->start` and `tw->end` in-place.** Since `tw` is stack-allocated in `CM_Trace` and this is a leaf function, this is safe; but the pattern would be dangerous if the call chain were ever refactored to reuse `tw` across multiple models.
