# code/bspc/l_poly.h

## File Purpose
Declares the interface for convex polygon (winding) operations used during BSP compilation. Windings represent convex polygons bounded by a set of 3D points and are the fundamental geometric primitive for CSG, clipping, and plane operations in the BSPC tool.

## Core Responsibilities
- Define the `winding_t` structure and associated limits/constants
- Declare allocation, deallocation, and memory-tracking functions for windings
- Declare geometric operations: area, center, bounds, plane extraction
- Declare clipping and chopping operations against planes
- Declare winding merging, reversing, and copying utilities
- Declare validation/error-checking functions for winding integrity
- Declare point-on-edge and plane-separation queries for BSP adjacency tests

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `winding_t` | struct | Convex polygon with a fixed-capacity point array (`p[4]`) that is variable-sized at allocation; holds `numpoints` and the vertex list |

## Global / File-Static State

None. (Memory tracking state is implicitly managed inside the implementation, exposed only via `WindingMemory()`, `WindingPeakMemory()`, `ActiveWindings()`.)

## Key Functions / Methods

### AllocWinding
- Signature: `winding_t *AllocWinding(int points)`
- Purpose: Allocate a winding for `points` vertices.
- Inputs: Vertex count.
- Outputs/Return: Pointer to a zeroed `winding_t`.
- Side effects: Heap allocation; updates internal memory/active counters.
- Calls: Not inferable from this file.
- Notes: Struct tail is variable-length beyond the declared `p[4]`.

### ClipWindingEpsilon
- Signature: `void ClipWindingEpsilon(winding_t *in, vec3_t normal, vec_t dist, vec_t epsilon, winding_t **front, winding_t **back)`
- Purpose: Split a winding by a plane, producing separate front and back fragments.
- Inputs: Input winding, plane normal+dist, tolerance epsilon.
- Outputs/Return: Front and back windings via out-pointers (either may be NULL).
- Side effects: Allocates new windings; does not free `in`.
- Calls: `AllocWinding` (implied).
- Notes: `ON_EPSILON` (0.1) is the default epsilon for point-on-plane classification.

### ChopWindingInPlace
- Signature: `void ChopWindingInPlace(winding_t **w, vec3_t normal, vec_t dist, vec_t epsilon)`
- Purpose: Clip a winding to the front side of a plane; free and replace if clipped.
- Inputs: Pointer-to-pointer winding, plane, epsilon.
- Outputs/Return: `*w` updated in place; set to NULL if entirely behind plane.
- Side effects: May free original, allocate replacement.
- Notes: Destructive variant of `ClipWindingEpsilon`.

### TryMergeWinding
- Signature: `winding_t *TryMergeWinding(winding_t *f1, winding_t *f2, vec3_t planenormal)`
- Purpose: Attempt to merge two co-planar windings sharing an edge into one convex winding.
- Inputs: Two windings, their shared plane normal.
- Outputs/Return: New merged winding, or NULL if merge is not valid.
- Side effects: Allocates result; originals are undisturbed.
- Notes: Leaves inputs intact — caller manages their lifetime.

### WindingError
- Signature: `int WindingError(winding_t *w)`
- Purpose: Validate a winding and return one of the `WE_*` error codes.
- Inputs: Winding to inspect.
- Outputs/Return: `WE_NONE` (0) on success; otherwise a `WE_*` flag.
- Side effects: None.
- Notes: Companion `WindingErrorString()` returns a human-readable message.

### FindPlaneSeperatingWindings
- Signature: `int FindPlaneSeperatingWindings(winding_t *w1, winding_t *w2, vec3_t dir, vec3_t normal, float *dist)`
- Purpose: Find a plane that separates two adjacent windings along their shared edge and a direction vector.
- Inputs: Two windings, a direction vector.
- Outputs/Return: Returns true if adjacent; fills `normal` and `dist`.
- Side effects: None.
- Notes: Used for AAS reachability and area-merging in the BSPC compiler.

**Notes on trivial helpers:** `WindingArea`, `WindingCenter`, `WindingBounds`, `WindingPlane`, `RemoveColinearPoints`, `RemoveEqualPoints`, `FreeWinding`, `CopyWinding`, `ReverseWinding`, `BaseWindingForPlane`, `ChopWinding`, `WindingOnPlaneSide`, `PointOnWinding`, `WindingsNonConvex`, `AddWindingPoint`, `pw`, `ResetWindings`, `WindingMemory`, `WindingPeakMemory`, `ActiveWindings` are standard geometric/diagnostic utilities whose names are self-describing.

## Control Flow Notes
This is a pure header — it participates in BSPC compile-time BSP construction. Windings are created during map parsing and brush splitting, clipped during CSG and portal generation, merged during area-merging passes, and freed after each phase completes. Not part of runtime game loop.

## External Dependencies
- Implicit dependency on `vec3_t`, `vec_t` from `mathlib.h` / `q_shared.h` (defined elsewhere).
- Implementation lives in `code/bspc/l_poly.c` (defined elsewhere).
- `MAX_POINTS_ON_WINDING` (96) caps the polygon vertex count during clipping operations.
