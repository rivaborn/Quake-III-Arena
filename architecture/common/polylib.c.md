# common/polylib.c

## File Purpose
Implements a convex polygon (winding) library used by the offline BSP compilation and map-processing tools. Provides allocation, clipping, geometric queries, and convex-hull merging operations on `winding_t` polygons.

## Core Responsibilities
- Allocate and free `winding_t` polygon objects with optional single-threaded diagnostics
- Clip windings against planes, producing front/back fragments (`ClipWindingEpsilon`, `ChopWindingInPlace`, `ChopWinding`)
- Generate a maximal base winding for an arbitrary plane (`BaseWindingForPlane`)
- Compute geometric properties: area, bounds, center, plane equation
- Validate winding geometry for convexity, planarity, and degeneracy (`CheckWinding`)
- Classify a winding relative to a plane (`WindingOnPlaneSide`)
- Incrementally grow a coplanar convex hull (`AddWindingToConvexHull`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `winding_t` | struct (typedef) | Convex polygon; `numpoints` + flexible `vec3_t p[4]` array (variable-size allocation) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `c_active_windings` | `int` | global | Current live winding count (single-threaded only) |
| `c_peak_windings` | `int` | global | High-water mark of simultaneous windings |
| `c_winding_allocs` | `int` | global | Total allocation count |
| `c_winding_points` | `int` | global | Total points allocated |
| `c_removed` | `int` | global | Points removed by `RemoveColinearPoints` |
| `numthreads` | `int` | global (extern) | Controls whether counters are updated; defined in threading layer |

## Key Functions / Methods

### AllocWinding
- Signature: `winding_t *AllocWinding(int points)`
- Purpose: Allocate a zeroed winding sized for `points` vertices.
- Inputs: `points` — vertex capacity.
- Outputs/Return: Pointer to new `winding_t`.
- Side effects: Bumps `c_winding_allocs`, `c_winding_points`, `c_active_windings`, `c_peak_windings` when `numthreads == 1`. Calls `malloc`.
- Calls: `malloc`, `memset`
- Notes: Size computed as `sizeof(vec_t)*3*points + sizeof(int)` — the `int` covers `numpoints`; `p` is variable-length.

### FreeWinding
- Signature: `void FreeWinding(winding_t *w)`
- Purpose: Free a winding, stamping it with `0xdeaddead` to detect double-free.
- Inputs: `w` — winding to free.
- Outputs/Return: void
- Side effects: Writes sentinel, decrements `c_active_windings`, calls `free`.
- Calls: `Error`, `free`

### ClipWindingEpsilon
- Signature: `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- Purpose: Split a winding by a plane into front and back fragments; points within `epsilon` are `SIDE_ON` and duplicated into both outputs.
- Inputs: `in` — source winding; `normal`/`dist` — clip plane; `epsilon` — tolerance.
- Outputs/Return: `*front`, `*back` — newly allocated windings (either may be NULL if all points are on one side).
- Side effects: Allocates two new windings via `AllocWinding`/`CopyWinding`. Does **not** free `in`.
- Calls: `DotProduct`, `CopyWinding`, `AllocWinding`, `Error`
- Notes: `dot` is `static` to work around a VC 4.2 optimizer bug. Generates interpolated split points when an edge crosses the plane.

### ChopWindingInPlace
- Signature: `void ChopWindingInPlace(winding_t **inout, vec3_t normal, vec_t dist, vec_t epsilon)`
- Purpose: Clip a winding to the front side of a plane in-place; frees the original if the back side is non-empty.
- Inputs: `*inout` — pointer to winding to modify; plane params.
- Outputs/Return: `*inout` updated to front fragment or NULL if fully clipped.
- Side effects: May free original winding and replace `*inout`.
- Calls: `DotProduct`, `AllocWinding`, `FreeWinding`, `Error`

### ChopWinding
- Signature: `winding_t *ChopWinding(winding_t *in, vec3_t normal, vec_t dist)`
- Purpose: Convenience wrapper; returns front fragment, frees both `in` and any back fragment.
- Inputs: `in` — winding (consumed); `normal`/`dist` — clip plane.
- Outputs/Return: Front winding or NULL.
- Side effects: Frees `in` and back winding.
- Calls: `ClipWindingEpsilon`, `FreeWinding`

### BaseWindingForPlane
- Signature: `winding_t *BaseWindingForPlane(vec3_t normal, vec_t dist)`
- Purpose: Create an axis-aligned quad of world-coordinate size that lies on the given plane — used as a starting winding for BSP brush faces.
- Inputs: Plane `normal` and `dist`.
- Outputs/Return: 4-point winding.
- Side effects: Allocates one winding.
- Calls: `AllocWinding`, `Error`, `DotProduct`, `VectorMA`, `VectorNormalize`, `CrossProduct`, `VectorScale`, `VectorSubtract`, `VectorAdd`

### AddWindingToConvexHull
- Signature: `void AddWindingToConvexHull(winding_t *w, winding_t **hull, vec3_t normal)`
- Purpose: Incrementally expand a coplanar convex hull to include all points of `w`.
- Inputs: `w` — winding whose points are added; `*hull` — current hull (NULL on first call); `normal` — shared plane normal.
- Outputs/Return: `*hull` updated.
- Side effects: Frees old hull and allocates a new one.
- Calls: `CopyWinding`, `VectorSubtract`, `VectorNormalize`, `CrossProduct`, `DotProduct`, `FreeWinding`, `AllocWinding`
- Notes: Uses a local `MAX_HULL_POINTS` (128) scratch buffer; finds back-to-front hull transitions to insert exterior points.

### Notes (minor functions)
- `RemoveColinearPoints`: removes points whose adjacent edges are nearly collinear (dot > 0.999); modifies winding in-place.
- `WindingPlane`, `WindingArea`, `WindingBounds`, `WindingCenter`: read-only geometric queries.
- `CheckWinding`: debug validator — calls `Error` on any geometry fault.
- `WindingOnPlaneSide`: classifies winding as FRONT/BACK/ON/CROSS relative to a plane.
- `pw`: debug print helper.

## Control Flow Notes
This file is a **tool-time** library (not compiled into the game runtime). It is used during BSP compilation (`q3map`, `bspc`) and the map editor (`q3radiant`). Functions are called during brush-to-BSP conversion: `BaseWindingForPlane` generates initial face polygons; `ClipWindingEpsilon`/`ChopWindingInPlace` trim them during node splitting; `AddWindingToConvexHull` is used in portal/area calculations. There is no frame or update loop involvement.

## External Dependencies
- `cmdlib.h` — `Error()`, `qboolean`, standard C includes
- `mathlib.h` — `vec_t`, `vec3_t`, all vector/cross/dot macros and functions, `SIDE_*` constants
- `polylib.h` — declares `winding_t`, `MAX_POINTS_ON_WINDING`, `ON_EPSILON`
- `qfiles.h` — `MAX_WORLD_COORD`, `MIN_WORLD_COORD`, `WORLD_SIZE`
- `numthreads` — **defined elsewhere** (threading layer, e.g. `common/threads.c`)
- `malloc`, `free`, `memset`, `memcpy`, `printf` — C standard library
