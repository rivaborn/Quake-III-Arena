# code/qcommon/cm_polylib.c

## File Purpose
Provides polygon (winding) geometry utilities used exclusively by the collision map (`cm_`) debug and visualization tools. It is not part of the runtime collision pipeline itself, only supporting diagnostic/visualization code.

## Core Responsibilities
- Allocate and free `winding_t` polygon objects with tracking counters
- Compute geometric properties: plane, area, bounds, center
- Clip windings against planes (both destructive and non-destructive variants)
- Validate winding convexity and planarity
- Classify winding position relative to a plane
- Construct a convex hull from multiple coplanar windings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `winding_t` | struct (defined in `cm_polylib.h`) | Variable-length polygon: `numpoints` + flexible `vec_t p[][3]` array of 3D vertices |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_active_windings` | `int` | global | Count of currently live winding allocations |
| `c_peak_windings` | `int` | global | High-water mark of concurrent windings |
| `c_winding_allocs` | `int` | global | Total allocation count |
| `c_winding_points` | `int` | global | Total points allocated across all windings |
| `c_removed` | `int` | global | Count of collinear points removed |

All counters are noted as unsafe for multi-threaded use.

## Key Functions / Methods

### AllocWinding
- **Signature:** `winding_t *AllocWinding(int points)`
- **Purpose:** Allocate a zeroed winding for `points` vertices; update debug counters.
- **Inputs:** `points` — desired vertex capacity
- **Outputs/Return:** Pointer to newly allocated `winding_t`
- **Side effects:** Increments `c_winding_allocs`, `c_winding_points`, `c_active_windings`; updates `c_peak_windings`
- **Calls:** `Z_Malloc`, `Com_Memset`
- **Notes:** Does not set `numpoints`; caller must assign it.

### FreeWinding
- **Signature:** `void FreeWinding(winding_t *w)`
- **Purpose:** Free a winding; poison the first word to detect double-frees.
- **Inputs:** `w` — winding to free
- **Outputs/Return:** void
- **Side effects:** Decrements `c_active_windings`; writes `0xdeaddead` sentinel
- **Calls:** `Com_Error`, `Z_Free`

### ClipWindingEpsilon
- **Signature:** `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- **Purpose:** Non-destructively split a winding by a plane into front/back fragments; generates interpolated split vertices.
- **Inputs:** `in` — source winding; `normal`/`dist` — clip plane; `epsilon` — on-plane tolerance
- **Outputs/Return:** `*front`, `*back` — newly allocated windings (either may be NULL)
- **Side effects:** Allocates up to two new windings via `AllocWinding`/`CopyWinding`
- **Calls:** `DotProduct`, `AllocWinding`, `CopyWinding`, `VectorCopy`, `Com_Error`
- **Notes:** `dot` is `static` to work around a VC 4.2 optimizer bug. Overrun is hard-faulted via `Com_Error`.

### ChopWindingInPlace
- **Signature:** `void ChopWindingInPlace(winding_t **inout, vec3_t normal, vec_t dist, vec_t epsilon)`
- **Purpose:** Destructively clip a winding to the front half-space; frees original and replaces pointer.
- **Inputs:** `*inout` — pointer to winding (freed on success); plane params
- **Outputs/Return:** `*inout` set to new front winding or NULL if fully clipped
- **Side effects:** Frees original winding; allocates replacement
- **Calls:** `FreeWinding`, `AllocWinding`, `Com_Error`

### ChopWinding
- **Signature:** `winding_t *ChopWinding(winding_t *in, vec3_t normal, vec_t dist)`
- **Purpose:** Return front fragment of `in` clipped by plane; frees original and back fragment.
- **Inputs:** Source winding + plane
- **Outputs/Return:** Front-side winding (may be NULL)
- **Side effects:** Frees `in` and back result
- **Calls:** `ClipWindingEpsilon`, `FreeWinding`

### AddWindingToConvexHull
- **Signature:** `void AddWindingToConvexHull(winding_t *w, winding_t **hull, vec3_t normal)`
- **Purpose:** Expand a coplanar convex hull to include all points of `w`; initializes hull from first call.
- **Inputs:** `w` — winding to merge; `*hull` — current hull (may be NULL); `normal` — shared plane normal
- **Outputs/Return:** `*hull` updated in place
- **Side effects:** Frees old hull; allocates new hull winding
- **Calls:** `CopyWinding`, `FreeWinding`, `AllocWinding`, `VectorSubtract`, `VectorNormalize2`, `CrossProduct`, `DotProduct`, `Com_Memcpy`
- **Notes:** Limited to `MAX_HULL_POINTS` (128). Works with local stack arrays for intermediate hull.

### Notes (minor helpers)
- `pw` — debug printf of winding vertices to stdout
- `RemoveColinearPoints` — in-place removal of collinear vertices using dot-product threshold 0.999
- `WindingPlane` / `WindingArea` / `WindingBounds` / `WindingCenter` — pure geometric queries, no side effects
- `BaseWindingForPlane` — creates a large axis-aligned quad on a plane as a starting clip winding
- `CopyWinding` / `ReverseWinding` — allocate copies with same or reversed vertex order
- `CheckWinding` — validation: asserts ≥3 points, nonzero area, all on-plane, all edges non-degenerate, convex
- `WindingOnPlaneSide` — classifies winding as `SIDE_FRONT/BACK/ON/CROSS`

## Control Flow Notes
This file is self-contained utility code. It has no frame/update/render role. It is pulled in by `cm_local.h` and consumed only by `cm_` debug/visualization functions, not by the production collision trace path.

## External Dependencies
- `cm_local.h` → transitively pulls in `q_shared.h`, `qcommon.h`, `cm_polylib.h`
- `Z_Malloc` / `Z_Free` — zone allocator (defined in `common.c`)
- `Com_Memset`, `Com_Memcpy`, `Com_Error` — common utilities (defined in `common.c`)
- `DotProduct`, `CrossProduct`, `VectorSubtract`, `VectorNormalize2`, `VectorMA`, `VectorScale`, `VectorLength` — math macros/functions from `q_shared.h`
- `SIDE_FRONT`, `SIDE_BACK`, `SIDE_ON`, `SIDE_CROSS`, `ON_EPSILON`, `MAX_MAP_BOUNDS`, `MAX_POINTS_ON_WINDING` — constants defined elsewhere (likely `cm_polylib.h` / `q_shared.h`)
