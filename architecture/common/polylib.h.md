# common/polylib.h

## File Purpose
Declares the `winding_t` polygon primitive and the full suite of convex-polygon (winding) utility functions used throughout Quake III's BSP compiler (`q3map`), collision system, and BSPC tool. Windings represent convex polygons defined by an ordered list of 3D vertices and are the fundamental geometric primitive for CSG, BSP splitting, and portal generation.

## Core Responsibilities
- Define the `winding_t` structure and its size limit constant
- Declare allocation and deallocation functions for windings
- Declare plane-clipping and chopping operations (the core BSP split primitive)
- Declare geometric query functions (area, bounds, plane, side classification)
- Declare convex hull merging support
- Define the `ON_EPSILON` tolerance used across all plane-side tests

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `winding_t` | struct | Convex polygon: vertex count + variable-length `vec3_t` array (declared as `p[4]` but heap-allocated to required size) |

## Global / File-Static State
None.

## Key Functions / Methods

### AllocWinding
- Signature: `winding_t *AllocWinding(int points)`
- Purpose: Heap-allocates a `winding_t` sized for `points` vertices.
- Inputs: `points` — desired vertex capacity.
- Outputs/Return: Pointer to newly allocated `winding_t`.
- Side effects: Heap allocation.
- Calls: Not inferable from this file.
- Notes: Callers must pair with `FreeWinding`. Struct tail is variable-length; the `p[4]` field is a C flexible-array idiom.

### ClipWindingEpsilon
- Signature: `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- Purpose: Splits a winding by a plane into front and back fragments with a configurable coplanarity epsilon.
- Inputs: `in` — source winding; `normal`/`dist` — splitting plane; `epsilon` — on-plane tolerance; `front`/`back` — output pointers (either may be NULL if all geometry is on one side).
- Outputs/Return: Populates `*front` and `*back`; either may be set to NULL.
- Side effects: Allocates new windings via `AllocWinding`.
- Calls: Not inferable from this file.
- Notes: Foundation of all BSP plane-splitting; `ON_EPSILON` (0.1) is the default tolerance used by higher-level callers.

### ChopWindingInPlace
- Signature: `void ChopWindingInPlace(winding_t **w, vec3_t normal, vec_t dist, vec_t epsilon)`
- Purpose: Clips winding to the front side of a plane, freeing the original and replacing `*w` with the clipped result (or NULL if fully clipped).
- Inputs: `w` — pointer-to-pointer to winding; `normal`/`dist` — clip plane; `epsilon` — tolerance.
- Outputs/Return: Modifies `*w` in place.
- Side effects: Frees original winding; may allocate replacement.
- Notes: Preferred over `ChopWinding` when the input is no longer needed; avoids a stale pointer.

### BaseWindingForPlane
- Signature: `winding_t *BaseWindingForPlane(vec3_t normal, vec_t dist)`
- Purpose: Creates a large initial winding covering an entire infinite plane, used to seed BSP portal generation.
- Inputs: Plane equation `(normal, dist)`.
- Outputs/Return: Newly allocated `winding_t` representing a large quad on the plane.
- Side effects: Heap allocation.

### WindingOnPlaneSide
- Signature: `int WindingOnPlaneSide(winding_t *w, vec3_t normal, vec_t dist)`
- Purpose: Classifies an entire winding as FRONT, BACK, ON, or CROSS relative to a plane.
- Inputs: Winding and plane equation.
- Outputs/Return: Integer side code (defined elsewhere, typically `SIDE_FRONT/BACK/ON/CROSS`).

### AddWindingToConvexHull
- Signature: `void AddWindingToConvexHull(winding_t *w, winding_t **hull, vec3_t normal)`
- Purpose: Merges a winding into a growing convex hull polygon (used for light volume or portal expansion).
- Inputs: `w` — winding to add; `hull` — accumulated hull (may be NULL initially); `normal` — plane normal for the hull.
- Side effects: May free and reallocate `*hull`.

### Notes
- `FreeWinding`, `CopyWinding`, `ReverseWinding`, `WindingArea`, `WindingCenter`, `WindingBounds`, `WindingPlane`, `RemoveColinearPoints`, `CheckWinding` are straightforward geometric helpers documented by name.
- `pw(winding_t *w)` is a debug-print helper (likely `PrintWinding`).
- `ChopWinding` is a non-in-place variant of `ChopWindingInPlace` that leaves the input intact.

## Control Flow Notes
This is a pure header — no control flow here. The declared functions are called during BSP compilation (CSG, portal generation, vis flooding) and at tool startup/shutdown for geometry construction. Not part of the runtime game loop.

## External Dependencies
- `vec3_t`, `vec_t` — defined in `mathlib.h` / `q_shared.h` (defined elsewhere).
- `MAX_POINTS_ON_WINDING` (64) constrains all winding allocations; callers must not exceed this.
- `ON_EPSILON` (0.1) — overridable at compile time via makefile `-D` flag.
