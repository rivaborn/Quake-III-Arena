# code/qcommon/cm_patch.h

## File Purpose
Defines the data structures and entry points for Quake III Arena's curved-surface (patch mesh) collision system. It bridges the patch tessellation pipeline with the broader collision model (`cm_`) subsystem.

## Core Responsibilities
- Declare types for patch collision geometry (`patchPlane_t`, `facet_t`, `patchCollide_t`, `cGrid_t`)
- Define capacity limits for facets and planes used during patch collision generation
- Expose the public entry point `CM_GeneratePatchCollide` to callers in the collision module
- Document known issues and design tradeoffs for curved-surface collision (via header comments)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `patchPlane_t` | struct | A single plane (4 floats) plus precomputed sign-bit mask for fast AABB-vs-plane classification |
| `facet_t` | struct | One triangular/quad facet of a patch: its surface plane index, border plane indices, inward flags, and no-adjust flags |
| `patchCollide_t` / `patchCollide_s` | struct | Complete collision representation of a patch mesh: bounds, flat arrays of planes and facets |
| `cGrid_t` | struct | Intermediate grid of 3D points used during patch subdivision before facet generation; supports optional wrap in width/height |

## Global / File-Static State

None.

## Key Functions / Methods

### CM_GeneratePatchCollide
- **Signature:** `struct patchCollide_s *CM_GeneratePatchCollide( int width, int height, vec3_t *points );`
- **Purpose:** Converts a raw grid of control points into a `patchCollide_t` suitable for runtime trace/position tests.
- **Inputs:** `width`, `height` — grid dimensions; `points` — flat array of `width * height` 3D points.
- **Outputs/Return:** Pointer to a heap-allocated `patchCollide_t`; ownership transferred to caller.
- **Side effects:** Allocates memory; defined in `cm_patch.c`.
- **Calls:** Not inferable from this header.
- **Notes:** Internally subdivides the grid to `SUBDIVIDE_DISTANCE` precision (≤16 units from curve) before building planes and facets.

### Notes on remaining entry points (declared in comment only)
- `CM_ClearLevelPatches` — frees patch collision data between map loads.
- `CM_TraceThroughPatchCollide` — sweeps a `traceWork_t` volume against a `patchCollide_t`.
- `CM_PositionTestInPatchCollide` — tests whether a volume already overlaps a patch.
- `CM_DrawDebugSurface` — visualization callback for debug rendering of patch collision geometry.

## Control Flow Notes
This header is consumed during map loading (`cm_load.c`) where patch surfaces are converted via `CM_GeneratePatchCollide`. The resulting `patchCollide_t` pointers are stored per-surface and queried every frame during `CM_Trace` / `CM_TransformedBoxTrace` calls, placing this data on the critical collision path for both server-side physics and client-side prediction.

## External Dependencies
- **Includes (implicit):** Relies on `cm_local.h` (or `qcommon.h`) for `vec3_t`, `qboolean`, `traceWork_t`.
- **Defined elsewhere:** `CM_ClearLevelPatches`, `CM_TraceThroughPatchCollide`, `CM_PositionTestInPatchCollide`, `CM_DrawDebugSurface` — all implemented in `cm_patch.c`. `traceWork_t` defined in `cm_local.h`.
