# code/qcommon/cm_trace.c

## File Purpose
Implements all collision trace and position-test logic for Quake III Arena's clip-map system. It sweeps axis-aligned bounding boxes (AABB), oriented capsules, and points through BSP trees and patch surfaces, returning the first solid intersection fraction and contact plane.

## Core Responsibilities
- Point/AABB/capsule position overlap tests against brushes, patches, and the BSP tree
- Swept-volume trace (AABB and capsule) through brushes and patch collide surfaces
- Capsule-vs-capsule and AABB-vs-capsule collision dispatch
- BSP tree traversal routing swept traces to the correct leaf nodes
- Coordinate transformation (rotation/translation) for traces against rotated sub-models
- Per-trace `traceWork_t` setup: symmetric sizing, signbit corner offsets, bounds, sphere params

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `traceWork_t` | struct (typedef) | All working state for a single trace: start/end, box size, offsets, bounds, result `trace_t`, sphere |
| `sphere_t` | struct (typedef) | Oriented capsule descriptor: radius, halfheight, offset vector, enable flag |
| `cbrush_t` | struct (typedef) | A convex brush: planes/sides, contents, AABB, checkcount |
| `cLeaf_t` | struct (typedef) | BSP leaf referencing brush and surface index lists |
| `cNode_t` | struct (typedef) | BSP interior node: splitting plane + two children |
| `cPatch_t` | struct (typedef) | Curved patch surface with a `patchCollide_s` structure |
| `leafList_t` | struct (typedef) | Accumulator for BSP leaf enumeration |
| `clipMap_t` | struct (typedef, extern `cm`) | The entire loaded clip-map: all nodes, leafs, brushes, surfaces |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cm` | `clipMap_t` | global (extern) | The active loaded clip-map used by all trace functions |
| `c_traces` | `int` | global (extern) | Trace call counter for statistics |
| `c_brush_traces` | `int` | global (extern) | Per-brush trace counter |
| `c_patch_traces` | `int` | global (extern) | Per-patch trace counter |
| `cm_noCurves` | `cvar_t *` | global (extern) | CVar disabling patch collision |

## Key Functions / Methods

### SquareRootFloat
- **Signature:** `float SquareRootFloat(float number)`
- **Purpose:** Fast approximate inverse-square-root (Quake fast rsqrt, 0x5f3759df) converted to a square root. Two Newton–Raphson refinements.
- **Inputs:** A non-negative float.
- **Outputs/Return:** Approximate `sqrt(number)`.
- **Side effects:** None.
- **Calls:** None (pure arithmetic).
- **Notes:** Used by capsule intersection math; accuracy sufficient for collision epsilon tolerances.

---

### CM_TestBoxInBrush
- **Signature:** `void CM_TestBoxInBrush(traceWork_t *tw, cbrush_t *brush)`
- **Purpose:** Tests whether the moving volume (AABB or sphere at `tw->start`) overlaps a brush. Sets `startsolid`/`allsolid` on hit.
- **Inputs:** `tw` — trace work state; `brush` — candidate brush.
- **Outputs/Return:** Void; mutates `tw->trace` on overlap.
- **Side effects:** Writes `tw->trace.startsolid`, `allsolid`, `fraction`, `contents`.
- **Calls:** `DotProduct`, `VectorSubtract`, `VectorAdd`.
- **Notes:** Skips first 6 axial planes for non-sphere case; axial AABB pretest applied first.

---

### CM_TestInLeaf
- **Signature:** `void CM_TestInLeaf(traceWork_t *tw, cLeaf_t *leaf)`
- **Purpose:** Tests the current position against all brushes and patches in a BSP leaf.
- **Inputs:** `tw`, `leaf`.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Side effects:** Increments `brush->checkcount` / `patch->checkcount` to deduplicate multi-leaf hits.
- **Calls:** `CM_TestBoxInBrush`, `CM_PositionTestInPatchCollide`.
- **Notes:** Patch testing guarded by `cm_noCurves->integer`.

---

### CM_TestCapsuleInCapsule
- **Signature:** `void CM_TestCapsuleInCapsule(traceWork_t *tw, clipHandle_t model)`
- **Purpose:** Positional overlap test between the trace capsule and a model capsule; checks all four sphere–sphere pairs and the cylinder overlap region.
- **Inputs:** `tw`, `model` handle.
- **Outputs/Return:** Void; sets `startsolid`/`allsolid`/`fraction` in `tw->trace`.
- **Side effects:** Writes `tw->trace`.
- **Calls:** `CM_ModelBounds`, `VectorLengthSquared`, `VectorSubtract`, `VectorAdd`, `VectorCopy`, `Square`.

---

### CM_TestBoundingBoxInCapsule
- **Signature:** `void CM_TestBoundingBoxInCapsule(traceWork_t *tw, clipHandle_t model)`
- **Purpose:** Positional overlap: transforms the problem by swapping roles — converts the AABB to a capsule and the capsule to a temporary box model, then delegates to `CM_TestInLeaf`.
- **Inputs:** `tw`, `model`.
- **Outputs/Return:** Void; mutates `tw`.
- **Side effects:** Modifies `tw->start`, `tw->end`, `tw->sphere` in-place.
- **Calls:** `CM_ModelBounds`, `CM_TempBoxModel`, `CM_ClipHandleToModel`, `CM_TestInLeaf`.

---

### CM_PositionTest
- **Signature:** `void CM_PositionTest(traceWork_t *tw)`
- **Purpose:** Full world position test: gathers all BSP leafs touching the volume, then tests each via `CM_TestInLeaf`.
- **Inputs:** `tw`.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Side effects:** Increments `cm.checkcount` twice; calls `CM_BoxLeafnums_r`.
- **Calls:** `CM_BoxLeafnums_r`, `CM_TestInLeaf`.
- **Notes:** `MAX_POSITION_LEAFS = 1024`.

---

### CM_TraceThroughBrush
- **Signature:** `void CM_TraceThroughBrush(traceWork_t *tw, cbrush_t *brush)`
- **Purpose:** Core slab-method sweep trace against a convex brush; finds enter/leave fractions across all planes and updates `tw->trace` if a nearer hit is found.
- **Inputs:** `tw`, `brush`.
- **Outputs/Return:** Void; mutates `tw->trace.fraction`, `plane`, `surfaceFlags`, `contents`.
- **Side effects:** Increments `c_brush_traces`.
- **Calls:** `DotProduct`, `VectorSubtract`, `VectorAdd`.
- **Notes:** Handles both sphere and AABB offset modes; uses `SURFACE_CLIP_EPSILON` for robustness.

---

### CM_TraceThroughLeaf
- **Signature:** `void CM_TraceThroughLeaf(traceWork_t *tw, cLeaf_t *leaf)`
- **Purpose:** Dispatches sweep traces to every brush and patch in a leaf.
- **Inputs:** `tw`, `leaf`.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Side effects:** Sets `b->checkcount`/`patch->checkcount`.
- **Calls:** `CM_TraceThroughBrush`, `CM_TraceThroughPatch`.

---

### CM_TraceThroughSphere
- **Signature:** `void CM_TraceThroughSphere(traceWork_t *tw, vec3_t origin, float radius, vec3_t start, vec3_t end)`
- **Purpose:** Ray–sphere intersection for capsule end-cap collision; solves quadratic, sets `tw->trace` on hit.
- **Inputs:** Sphere `origin`, `radius`, ray `start`/`end`.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Calls:** `VectorNormalize`, `CM_DistanceFromLineSquared`, `SquareRootFloat`, `VectorMA`, `DotProduct`.
- **Notes:** Sets `tw->trace.contents = CONTENTS_BODY`.

---

### CM_TraceThroughVerticalCylinder
- **Signature:** `void CM_TraceThroughVerticalCylinder(traceWork_t *tw, vec3_t origin, float radius, float halfheight, vec3_t start, vec3_t end)`
- **Purpose:** Ray–infinite-cylinder intersection (2D projection), clipped to `halfheight`; used for capsule body segment.
- **Inputs:** Cylinder `origin`, `radius`, `halfheight`, ray endpoints.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Calls:** `SquareRootFloat`, `CM_DistanceFromLineSquared`, `VectorNormalize`, `VectorMA`, `DotProduct`.

---

### CM_TraceCapsuleThroughCapsule
- **Signature:** `void CM_TraceCapsuleThroughCapsule(traceWork_t *tw, clipHandle_t model)`
- **Purpose:** Full capsule-vs-capsule swept collision: tests cylinder body then both sphere end-caps.
- **Inputs:** `tw`, `model`.
- **Outputs/Return:** Void; mutates `tw->trace`.
- **Calls:** `CM_ModelBounds`, `CM_TraceThroughVerticalCylinder`, `CM_TraceThroughSphere`.

---

### CM_TraceThroughTree
- **Signature:** `void CM_TraceThroughTree(traceWork_t *tw, int num, float p1f, float p2f, vec3_t p1, vec3_t p2)`
- **Purpose:** Recursive BSP traversal; splits the trace segment at each node plane and descends both sides as needed.
- **Inputs:** Node index `num` (negative = leaf), fractional interval `[p1f, p2f]`, world positions `p1`/`p2`.
- **Outputs/Return:** Void; mutates `tw->trace` via leaf dispatch.
- **Side effects:** None beyond `tw->trace`.
- **Calls:** `CM_TraceThroughLeaf` (at leaves), itself recursively.
- **Notes:** Uses `offset = 2048` for non-axial, non-point traces (acknowledged as "silly" in source).

---

### CM_Trace
- **Signature:** `void CM_Trace(trace_t *results, const vec3_t start, const vec3_t end, vec3_t mins, vec3_t maxs, clipHandle_t model, const vec3_t origin, int brushmask, int capsule, sphere_t *sphere)`
- **Purpose:** Primary entry point. Initialises `traceWork_t`, handles position vs. sweep dispatch, and selects the correct collision mode (AABB, capsule, capsule-vs-capsule, world tree).
- **Inputs:** Start/end, volume mins/maxs, model handle, origin, content mask, capsule flag, optional pre-built sphere.
- **Outputs/Return:** Void; writes `*results`.
- **Side effects:** Increments `cm.checkcount`, `c_traces`; writes `results`.
- **Calls:** `CM_ClipHandleToModel`, `CM_PositionTest`, `CM_TestInLeaf`, `CM_TestCapsuleInCapsule`, `CM_TestBoundingBoxInCapsule`, `CM_TraceThroughTree`, `CM_TraceThroughLeaf`, `CM_TraceCapsuleThroughCapsule`, `CM_TraceBoundingBoxThroughCapsule`.
- **Notes:** Asserts unit-length contact normal on valid hits; symmetric AABB offset applied before any dispatch.

---

### CM_BoxTrace
- **Signature:** `void CM_BoxTrace(trace_t *results, const vec3_t start, const vec3_t end, vec3_t mins, vec3_t maxs, clipHandle_t model, int brushmask, int capsule)`
- **Purpose:** Thin wrapper; calls `CM_Trace` with zero origin and no pre-built sphere.
- **Calls:** `CM_Trace`.

---

### CM_TransformedBoxTrace
- **Signature:** `void CM_TransformedBoxTrace(trace_t *results, ..., const vec3_t origin, const vec3_t angles, int capsule)`
- **Purpose:** Traces against a rotated/translated sub-model by transforming start/end into model-local space, running `CM_Trace`, then rotating the contact plane back to world space.
- **Inputs:** As `CM_BoxTrace` plus `origin` and `angles`.
- **Outputs/Return:** Void; writes `*results` with recalculated `endpos`.
- **Side effects:** Builds rotation matrix; modifies local copies only.
- **Calls:** `CreateRotationMatrix`, `RotatePoint`, `TransposeMatrix`, `CM_Trace`.
- **Notes:** Comment acknowledges AABB rotation is still approximate; capsule rotation is exact.

## Control Flow Notes
This file is used during both physics/game simulation frames and server-side entity clipping. `CM_BoxTrace` / `CM_TransformedBoxTrace` are called from `SV_Trace` / `CG_Trace` every frame for all movement, projectile, and visibility queries. There is no init or shutdown logic here; the `cm` global is populated by `cm_load.c` before any trace is issued.

## External Dependencies
- **`cm_local.h`** — all type definitions (`traceWork_t`, `cbrush_t`, `cLeaf_t`, `clipMap_t`, `sphere_t`, etc.), extern declarations, and `SURFACE_CLIP_EPSILON`
- **`q_shared.h`** (via `cm_local.h`) — `vec3_t`, `trace_t`, `cplane_t`, `qboolean`, `VectorCopy`, `DotProduct`, `VectorMA`, `VectorNormalize`, `AngleVectors`, `Square`, `CONTENTS_BODY`, etc.
- **`cm_patch.c`** — `CM_TraceThroughPatchCollide`, `CM_PositionTestInPatchCollide` (defined elsewhere)
- **`cm_test.c` / `cm_load.c`** — `CM_BoxLeafnums_r`, `CM_StoreLeafs`, `CM_ClipHandleToModel`, `CM_ModelBounds`, `CM_TempBoxModel` (defined elsewhere)
- **`c_traces`, `c_brush_traces`, `c_patch_traces`** — statistic counters defined in `cm_load.c`
