# code/bspc/l_poly.c

## File Purpose
Implements convex polygon (winding) operations for the BSPC map compiler tool. Provides the full lifecycle—allocation, clipping, merging, validation, and geometry queries—for `winding_t` structures used during BSP construction and AAS area generation.

## Core Responsibilities
- Allocate and free `winding_t` objects with optional single-threaded usage statistics
- Generate a large base winding from a plane definition
- Clip windings against planes (epsilon-tolerant, in-place and copy variants)
- Merge two adjacent windings sharing a common edge (both convex-safe and brute-force variants)
- Query winding geometry: area, center, bounds, plane, side classification
- Validate winding integrity and report structured error codes
- Remove degenerate geometry: colinear points, duplicate/equal points

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `winding_t` | struct | Convex polygon: `numpoints` + variable-length `vec3_t p[4]` point array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_active_windings` | `int` | global (file) | Count of currently live windings (single-thread only) |
| `c_peak_windings` | `int` | global (file) | High-water mark of simultaneous windings |
| `c_winding_allocs` | `int` | global (file) | Total allocation count |
| `c_winding_points` | `int` | global (file) | Total points allocated |
| `c_windingmemory` | `int` | global (file) | Current winding memory usage in bytes |
| `c_peak_windingmemory` | `int` | global (file) | Peak winding memory usage |
| `windingerror` | `char[1024]` | global (file) | Last error string from `WindingError` |
| `c_removed` | `int` | global (file) | Count of colinear points removed |
| `numthreads` | `int` | extern global | Thread count; gates stat updates to single-threaded mode |

## Key Functions / Methods

### AllocWinding
- **Signature:** `winding_t *AllocWinding(int points)`
- **Purpose:** Allocates a zeroed winding for `points` vertices.
- **Inputs:** `points` — desired vertex capacity
- **Outputs/Return:** Pointer to new `winding_t`
- **Side effects:** Updates `c_winding_allocs`, `c_winding_points`, `c_active_windings`, `c_peak_windings`, `c_windingmemory`, `c_peak_windingmemory` when single-threaded
- **Calls:** `GetMemory`, `memset`, `MemorySize`
- **Notes:** Size is `sizeof(vec_t)*3*points + sizeof(int)`; the `p` array is variable-length past the struct

### FreeWinding
- **Signature:** `void FreeWinding(winding_t *w)`
- **Purpose:** Frees a winding and poisons its memory with `0xdeaddead`.
- **Side effects:** Decrements `c_active_windings`, `c_windingmemory`; calls `FreeMemory`
- **Notes:** Guards against double-free via sentinel check

### BaseWindingForPlane
- **Signature:** `winding_t *BaseWindingForPlane(vec3_t normal, vec_t dist)`
- **Purpose:** Creates an oversized 4-point winding covering the infinite extent of a plane, suitable as a starting clip polygon.
- **Inputs:** Plane `normal` and `dist`
- **Outputs/Return:** Allocated 4-point `winding_t`
- **Notes:** Uses `BOGUS_RANGE` (65535) to project a large axis-aligned quad

### ClipWindingEpsilon
- **Signature:** `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- **Purpose:** Clips winding `in` against a plane, producing `front` and `back` fragments. Does not free the input.
- **Inputs:** Input winding, plane (normal+dist), epsilon for on-plane tolerance
- **Outputs/Return:** `*front` and `*back` (either may be NULL if all points on one side)
- **Side effects:** Allocates up to two new windings
- **Notes:** Computes split points with axis-aligned normal fast-paths to minimize float error; comment warns against static locals due to multithreading

### ChopWindingInPlace
- **Signature:** `void ChopWindingInPlace(winding_t **inout, vec3_t normal, vec_t dist, vec_t epsilon)`
- **Purpose:** In-place front-side clip; frees original if fully clipped away or replaces pointer with trimmed winding.
- **Side effects:** May free `*inout` and replace with new allocation or NULL

### ChopWinding
- **Signature:** `winding_t *ChopWinding(winding_t *in, vec3_t normal, vec_t dist)`
- **Purpose:** Returns the front fragment of `in` clipped by the plane; frees `in` and the back fragment.
- **Calls:** `ClipWindingEpsilon`, `FreeWinding`

### TryMergeWinding
- **Signature:** `winding_t *TryMergeWinding(winding_t *f1, winding_t *f2, vec3_t planenormal)`
- **Purpose:** Merges two coplanar windings sharing an edge only if the result remains convex; originals are NOT freed.
- **Outputs/Return:** New merged winding or NULL on failure
- **Notes:** `CONTINUOUS_EPSILON` = 0.005; checks convexity at both junction points

### MergeWindings
- **Signature:** `winding_t *MergeWindings(winding_t *w1, winding_t *w2, vec3_t planenormal)`
- **Purpose:** Brute-force union of two windings into the smallest enclosing convex polygon.
- **Calls:** `RemoveEqualPoints`, `Log_Print`, `AllocWinding`, `RemoveColinearPoints`
- **Notes:** Inserts w2 points into w1's point list by computing separating edge normals; more aggressive than `TryMergeWinding`

### WindingError
- **Signature:** `int WindingError(winding_t *w)`
- **Purpose:** Validates a winding and returns a `WE_*` error code; stores a human-readable string in `windingerror`.
- **Outputs/Return:** `WE_NONE` (0) or one of `WE_NOTENOUGHPOINTS`, `WE_SMALLAREA`, `WE_POINTBOGUSRANGE`, `WE_POINTOFFPLANE`, `WE_DEGENERATEEDGE`, `WE_NONCONVEX`

### FindPlaneSeperatingWindings
- **Signature:** `int FindPlaneSeperatingWindings(winding_t *w1, winding_t *w2, vec3_t dir, vec3_t normal, float *dist)`
- **Purpose:** Finds a plane (containing `dir`) that separates the two windings, indicating adjacency along a shared edge.
- **Outputs/Return:** `true` if a valid separating plane was found, writing `normal` and `dist`

### Notes (minor helpers)
- `pw` — debug print of winding points to stdout
- `ResetWindings` — zero all counters and clear error string
- `WindingArea`, `WindingBounds`, `WindingCenter`, `WindingPlane` — pure geometric queries
- `RemoveColinearPoints`, `RemoveEqualPoints` — degenerate point removal
- `AddWindingPoint`, `PointOnWinding` — point insertion/membership tests
- `WindingsNonConvex` — convexity test between two adjacent polygons
- `CheckWinding` — assert-style validation (calls `Error` on failure, unlike `WindingError`)

## Control Flow Notes
This file is a utility library used during BSP compilation (`bspc`). It is called during the brush-splitting and AAS area generation passes—not during game runtime. There is no frame loop involvement; functions are invoked on demand by higher-level BSP/AAS construction code.

## External Dependencies
- `<malloc.h>` — system allocation (underlying `GetMemory`)
- `l_cmd.h` — `Error`, `qboolean`, `vec_t` primitives
- `l_math.h` — `vec3_t`, `DotProduct`, `CrossProduct`, `VectorNormalize`, `VectorLength`, etc.
- `l_log.h` — `Log_Print`, `Log_Write` for degenerate-case diagnostics
- `l_mem.h` — `GetMemory`, `FreeMemory`, `MemorySize` (custom allocator with size tracking)
- `numthreads` — extern from the BSPC threading system (defined elsewhere)
- `vec3_origin` — extern zero-vector (defined in `l_math.c`)
