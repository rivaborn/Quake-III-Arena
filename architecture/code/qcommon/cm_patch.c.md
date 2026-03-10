# code/qcommon/cm_patch.c

## File Purpose
Implements collision detection for quadratic Bezier patch meshes (curved surfaces) in Quake III Arena. It converts a patch control point grid into a flat facet/plane representation, then provides trace and position-test entry points used by the collision model system.

## Core Responsibilities
- Subdivide a quadratic Bezier grid until all curve segments are within `SUBDIVIDE_DISTANCE` of linear
- Remove degenerate (duplicate) columns and transpose the grid to subdivide both axes
- Build a plane list and facet list (`patchCollide_t`) from the subdivided grid triangles
- Add axial and edge bevel planes to each facet to prevent tunneling
- Perform swept-volume and point traces against a `patchCollide_t`
- Perform static position (overlap) tests against a `patchCollide_t`
- Render debug geometry for patch collision surfaces via a callback

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `patchPlane_t` | struct | A plane (float[4]) plus precomputed signbits for AABB offset lookup |
| `facet_t` | struct | One triangular or quad collision face: surface plane index + border plane indices/inward flags |
| `patchCollide_t` / `patchCollide_s` | struct | Final collision shape: bounds, plane array, facet array |
| `cGrid_t` | struct | Working grid of 3D points (up to 129×129) used during subdivision |
| `edgeName_t` | enum | `EN_TOP/RIGHT/BOTTOM/LEFT` — symbolic indices for the four quad borders |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `c_totalPatchBlocks` | `int` | global | Stat counter: total grid quad blocks processed |
| `c_totalPatchSurfaces` | `int` | global | Stat counter: total patch surfaces generated |
| `c_totalPatchEdges` | `int` | global | Stat counter: total patch edges processed |
| `debugPatchCollide` | `const patchCollide_t *` | static | Points to last-hit patch collide for debug rendering |
| `debugFacet` | `const facet_t *` | static | Points to last-hit facet for debug rendering |
| `debugBlock` | `qboolean` | static | Flags that a bisecting-border block was encountered |
| `debugBlockPoints` | `vec3_t[4]` | static | Corner points of the debug block quad |
| `numPlanes` | `int` | static (file) | Running count during `CM_PatchCollideFromGrid` |
| `planes` | `patchPlane_t[MAX_PATCH_PLANES]` | static (file) | Scratch plane table during generation |
| `numFacets` | `int` | static (file) | Running count during `CM_PatchCollideFromGrid` |
| `facets` | `facet_t[MAX_PATCH_PLANES]` | static (file) | Scratch facet table during generation |

## Key Functions / Methods

### CM_ClearLevelPatches
- Signature: `void CM_ClearLevelPatches( void )`
- Purpose: Resets debug pointers between level loads.
- Inputs: none
- Outputs/Return: void
- Side effects: Nulls `debugPatchCollide`, `debugFacet`
- Calls: nothing

### CM_GeneratePatchCollide
- Signature: `struct patchCollide_s *CM_GeneratePatchCollide( int width, int height, vec3_t *points )`
- Purpose: Main entry point — takes raw control-point grid, subdivides it, and produces an allocated `patchCollide_t` ready for tracing.
- Inputs: `width`, `height` (must be odd, ≥3, ≤`MAX_GRID_SIZE`); `points` packed row-major
- Outputs/Return: Hunk-allocated `patchCollide_t *`
- Side effects: Allocates from hunk (`h_high`); increments `c_totalPatchBlocks`
- Calls: `CM_SetGridWrapWidth`, `CM_SubdivideGridColumns`, `CM_RemoveDegenerateColumns`, `CM_TransposeGrid`, `AddPointToBounds`, `ClearBounds`, `CM_PatchCollideFromGrid`, `Hunk_Alloc`, `Com_Error`
- Notes: Bounds are expanded by 1 unit after generation for epsilon safety.

### CM_PatchCollideFromGrid
- Signature: `static void CM_PatchCollideFromGrid( cGrid_t *grid, patchCollide_t *pf )`
- Purpose: Iterates all quad cells of the subdivided grid, builds triangle planes, assigns border planes, validates and bevel-extends each facet, then copies results into `pf`.
- Inputs: fully-subdivided `grid`, output struct `pf`
- Outputs/Return: void; populates `pf->planes`, `pf->facets`, `pf->numPlanes`, `pf->numFacets`
- Side effects: Writes file-static `planes[]`/`facets[]`; allocates via `Hunk_Alloc`
- Calls: `CM_FindPlane`, `CM_EdgePlaneNum`, `CM_SetBorderInward`, `CM_ValidateFacet`, `CM_AddFacetBevels`, `Com_Error`, `Com_Memset`, `Com_Memcpy`, `Hunk_Alloc`

### CM_TraceThroughPatchCollide
- Signature: `void CM_TraceThroughPatchCollide( traceWork_t *tw, const struct patchCollide_s *pc )`
- Purpose: Swept-volume (box or capsule) trace against all facets; updates `tw->trace` if a closer hit is found.
- Inputs: `tw` (trace work with start/end/shape), `pc` (patch collide)
- Outputs/Return: void; mutates `tw->trace.fraction` and `tw->trace.plane`
- Side effects: May set `debugPatchCollide`/`debugFacet`; reads cvar `r_debugSurfaceUpdate`
- Calls: `CM_TracePointThroughPatchCollide` (for point traces), `CM_CheckFacetPlane`, `Cvar_Get`

### CM_TracePointThroughPatchCollide
- Signature: `void CM_TracePointThroughPatchCollide( traceWork_t *tw, const struct patchCollide_s *pc )`
- Purpose: Optimized point-only trace (no volume); uses intersection parametrics per plane.
- Inputs: `tw` (must have `isPoint` true and `cm_playerCurveClip` enabled), `pc`
- Outputs/Return: void; mutates `tw->trace`
- Side effects: Sets debug statics; reads `cm_playerCurveClip` cvar
- Notes: Returns early if `!tw->isPoint` or curve clip is disabled.

### CM_PositionTestInPatchCollide
- Signature: `qboolean CM_PositionTestInPatchCollide( traceWork_t *tw, const struct patchCollide_s *pc )`
- Purpose: Tests whether a volume (AABB or capsule) is currently overlapping any facet.
- Inputs: `tw`, `pc`
- Outputs/Return: `qtrue` if inside any facet, else `qfalse`
- Notes: Returns `qfalse` immediately for point traces (surfaces have no volume).

### CM_AddFacetBevels
- Signature: `void CM_AddFacetBevels( facet_t *facet )`
- Purpose: Adds axial bevel planes and non-axial edge bevel planes to prevent a swept box from slipping through crack edges.
- Side effects: Extends `facet->borderPlanes[]`, increments `facet->numBorders`; allocates/frees temporary windings
- Calls: `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `CM_PlaneEqual`, `CM_FindPlane2`, `CopyWinding`, `FreeWinding`, `CM_SnapVector`

### CM_DrawDebugSurface
- Signature: `void CM_DrawDebugSurface( void (*drawPoly)(int color, int numPoints, float *points) )`
- Purpose: Renderer-called debug visualizer; draws all facet planes of the last-hit patch collide using the supplied polygon callback.
- Side effects: Reads cvars `r_debugSurface`, `cm_debugSize`; calls `BotDrawDebugPolygons` when `r_debugSurface != 1`

**Notes on trivial helpers:** `CM_SignbitsForNormal`, `CM_PlaneFromPoints`, `CM_NeedsSubdivision`, `CM_Subdivide`, `CM_TransposeGrid`, `CM_SetGridWrapWidth`, `CM_SubdivideGridColumns`, `CM_ComparePoints`, `CM_RemoveDegenerateColumns`, `CM_PlaneEqual`, `CM_SnapVector`, `CM_FindPlane`/`CM_FindPlane2`, `CM_PointOnPlaneSide`, `CM_GridPlane`, `CM_EdgePlaneNum`, `CM_SetBorderInward`, `CM_ValidateFacet`, `CM_CheckFacetPlane` are all internal geometry utilities supporting the pipeline above.

## Control Flow Notes
- **Load time:** `cm_load.c` calls `CM_GeneratePatchCollide` for each patch surface, storing the result in `cPatch_t::pc`.
- **Trace time (per frame):** `cm_trace.c` calls `CM_TraceThroughPatchCollide` and `CM_PositionTestInPatchCollide` for every patch leaf the trace touches.
- **Level shutdown:** `CM_ClearLevelPatches` is called to reset debug pointers; actual memory is freed with the hunk.
- **Render debug:** `CM_DrawDebugSurface` is called by the renderer when `r_debugSurface 1`.

## External Dependencies
- `cm_local.h` → `q_shared.h`, `qcommon.h`, `cm_polylib.h` (winding utilities: `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `FreeWinding`, `CopyWinding`)
- `cm_patch.h` — type and constant definitions (`patchPlane_t`, `facet_t`, `patchCollide_t`, `cGrid_t`, `MAX_FACETS`, `MAX_PATCH_PLANES`, `MAX_GRID_SIZE`, `SUBDIVIDE_DISTANCE`, etc.)
- **Defined elsewhere:** `Hunk_Alloc`, `Com_Error`, `Com_Printf`, `Com_DPrintf`, `Com_Memset`, `Com_Memcpy`, `Cvar_Get`, `VectorMA`, `DotProduct`, `CrossProduct`, `VectorNormalize`, `VectorNegate`, `VectorSubtract`, `VectorAdd`, `VectorCopy`, `VectorClear`, `Vector4Copy`, `AddPointToBounds`, `ClearBounds`, `cm_playerCurveClip`, `BotDrawDebugPolygons`
