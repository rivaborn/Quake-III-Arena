# code/qcommon/cm_polylib.h

## File Purpose
Declares the `winding_t` polygon type and its associated operations for convex polygon manipulation. Used exclusively by the collision model debug/visualization subsystem (`cm_` functions), not by general gameplay or rendering.

## Core Responsibilities
- Define the `winding_t` convex polygon primitive
- Declare allocation, copy, and deallocation routines for windings
- Declare geometric query operations (area, center, bounds, plane, side classification)
- Declare clipping and chopping operations against planes
- Declare convex hull construction helper
- Define plane-side and clipping constants

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `winding_t` | struct | Convex polygon with a fixed-capacity point array (`p[4]`, variable-sized via allocation); the `4` is a placeholder — actual size is determined at alloc time |

## Global / File-Static State
None.

## Key Functions / Methods

### AllocWinding
- Signature: `winding_t *AllocWinding(int points)`
- Purpose: Heap-allocate a winding large enough to hold `points` vertices.
- Inputs: `points` — desired vertex capacity.
- Outputs/Return: Pointer to a newly allocated `winding_t`.
- Side effects: Heap allocation.
- Calls: Defined elsewhere (`cm_polylib.c`).
- Notes: Caller is responsible for freeing with `FreeWinding`.

### ClipWindingEpsilon
- Signature: `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- Purpose: Splits a winding by a plane with a configurable epsilon tolerance, producing front and back fragments.
- Inputs: Input winding, plane normal/dist, epsilon threshold.
- Outputs/Return: `*front` and `*back` set to new windings (either may be NULL).
- Side effects: Allocates new windings; does not free `in`.
- Notes: Use `CLIP_EPSILON` (0.1) for standard tolerance.

### ChopWindingInPlace
- Signature: `void ChopWindingInPlace(winding_t **w, vec3_t normal, vec_t dist, vec_t epsilon)`
- Purpose: Clips a winding to the front side of a plane, freeing the original if it is clipped away or replaced.
- Inputs: Pointer-to-pointer to winding, plane normal/dist, epsilon.
- Outputs/Return: `*w` updated in place; may be set to NULL if fully clipped.
- Side effects: Frees original winding if a new one is produced or fully clipped.

### WindingOnPlaneSide
- Signature: `int WindingOnPlaneSide(winding_t *w, vec3_t normal, vec_t dist)`
- Purpose: Classifies all vertices of a winding relative to a plane.
- Outputs/Return: One of `SIDE_FRONT`, `SIDE_BACK`, `SIDE_ON`, or `SIDE_CROSS`.

### AddWindingToConvexHull
- Signature: `void AddWindingToConvexHull(winding_t *w, winding_t **hull, vec3_t normal)`
- Purpose: Merges a winding into an existing convex hull polygon (used in debug visualization).
- Side effects: May allocate a new hull winding and free the old one.

### Notes
- `pw`, `WindingArea`, `WindingCenter`, `WindingBounds`, `WindingPlane`, `CheckWinding`, `RemoveColinearPoints`, `BaseWindingForPlane`, `CopyWinding`, `ReverseWinding`, `FreeWinding`, and `ChopWinding` are straightforward geometric utilities; all are defined in `cm_polylib.c`.

## Control Flow Notes
This header is included only within `cm_` debug/visualization code. It is not part of the main collision detection pipeline (traces, patch collisions) and has no direct role in the per-frame game loop. It is used when developer debug rendering of collision geometry is active.

## External Dependencies
- `vec3_t`, `vec_t` — defined in `q_shared.h` / `qcommon.h`
- All function bodies defined in `code/qcommon/cm_polylib.c`
- `MAX_POINTS_ON_WINDING`, `ON_EPSILON`, `CLIP_EPSILON` constants are self-contained in this header
