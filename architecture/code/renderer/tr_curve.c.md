# code/renderer/tr_curve.c

## File Purpose
Converts raw Bézier patch control-point grids (read from map data) into subdivided `srfGridMesh_t` render surfaces. It handles adaptive LOD subdivision, normal generation, patch stitching via column/row insertion, and mesh lifecycle (alloc/free).

## Core Responsibilities
- Adaptively subdivide a patch mesh in both axes based on `r_subdivisions` error tolerance
- Compute per-vertex normals accounting for mesh wrapping and degenerate edges
- Cull collinear rows/columns from the final grid
- Allocate and populate `srfGridMesh_t` with LOD error tables and bounding data
- Free `srfGridMesh_t` allocations (supports `PATCH_STITCHING` heap path)
- Insert a new column or row into an existing grid (patch stitching)
- Optionally transpose grid for longer triangle strips

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `srfGridMesh_t` | struct (typedef) | Render-ready grid surface: verts, LOD error arrays, bounds, surface type tag |
| `drawVert_t` | struct (typedef, defined in qfiles.h) | Per-vertex data: position, UV, lightmap UV, color, normal |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ctrl` (in `R_SubdividePatchToGrid`) | `MAC_STATIC drawVert_t[MAX_GRID_SIZE][MAX_GRID_SIZE]` | static local | Working grid buffer; `MAC_STATIC` avoids large stack alloc on Mac |
| `ctrl` (in `R_GridInsertColumn`, `R_GridInsertRow`) | `MAC_STATIC drawVert_t[MAX_GRID_SIZE][MAX_GRID_SIZE]` | static local | Same pattern for grid mutation helpers |

## Key Functions / Methods

### LerpDrawVert
- **Signature:** `static void LerpDrawVert(drawVert_t *a, drawVert_t *b, drawVert_t *out)`
- **Purpose:** Linearly interpolates all fields of two draw vertices at t=0.5.
- **Inputs:** Two source vertices `a`, `b`.
- **Outputs/Return:** Writes result into `out` (may alias `a` or `b`).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Color channels use integer `>> 1` (not float), avoids rounding bias at cost of minor precision.

### MakeMeshNormals
- **Signature:** `static void MakeMeshNormals(int width, int height, drawVert_t ctrl[MAX_GRID_SIZE][MAX_GRID_SIZE])`
- **Purpose:** Computes smooth vertex normals by accumulating cross-products from up to 8 neighbors at distances 1–3, handling cylindrical/toroidal wrap detection.
- **Inputs:** Grid dimensions and `ctrl` array.
- **Outputs/Return:** Writes `dv->normal` for every vertex in-place.
- **Side effects:** Modifies `ctrl[*][*].normal`.
- **Calls:** `VectorSubtract`, `VectorLengthSquared`, `VectorNormalize2`, `CrossProduct`, `VectorAdd`, `VectorClear`, `VectorCopy`.
- **Notes:** `wrapWidth`/`wrapHeight` detected by comparing opposite-edge positions (threshold 1.0 squared). If no valid neighbor cross-products exist, `count` is clamped to 1 to avoid divide-by-zero (sum will be zero, producing a zero normal — guarded with commented debug print).

### R_CreateSurfaceGridMesh
- **Signature:** `srfGridMesh_t *R_CreateSurfaceGridMesh(int width, int height, drawVert_t ctrl[MAX_GRID_SIZE][MAX_GRID_SIZE], float errorTable[2][MAX_GRID_SIZE])`
- **Purpose:** Allocates and populates a `srfGridMesh_t` from a finalized control grid, including LOD error tables and bounding sphere/box.
- **Inputs:** Grid dims, control verts, error tables.
- **Outputs/Return:** Heap-allocated `srfGridMesh_t *`.
- **Side effects:** Allocates via `ri.Malloc` (PATCH_STITCHING path) or `ri.Hunk_Alloc`. Sets `surfaceType = SF_GRID`.
- **Calls:** `ri.Malloc` / `ri.Hunk_Alloc`, `Com_Memset`, `Com_Memcpy`, `ClearBounds`, `AddPointToBounds`, `VectorAdd`, `VectorScale`, `VectorSubtract`, `VectorLength`, `VectorCopy`.
- **Notes:** `PATCH_STITCHING` conditional swaps hunk for heap allocations so grids can be freed and replaced during stitching. `verts[1]` is a flexible-array-style trailing allocation (size accounts for `width*height - 1` extra verts).

### R_FreeSurfaceGridMesh
- **Signature:** `void R_FreeSurfaceGridMesh(srfGridMesh_t *grid)`
- **Purpose:** Frees the three heap blocks comprising a stitchable grid mesh.
- **Side effects:** Calls `ri.Free` three times.

### R_SubdividePatchToGrid
- **Signature:** `srfGridMesh_t *R_SubdividePatchToGrid(int width, int height, drawVert_t points[MAX_PATCH_SIZE*MAX_PATCH_SIZE])`
- **Purpose:** Main entry point. Adaptively subdivides a Bézier patch in both U and V, culls collinear rows/columns, optionally transposes for strip optimization, computes normals, and returns a render-ready grid.
- **Inputs:** Raw patch dimensions (must be odd ≥ 3) and flat control-point array.
- **Outputs/Return:** Newly allocated `srfGridMesh_t *`.
- **Side effects:** Reads `r_subdivisions->value`. Uses `MAC_STATIC` local buffers.
- **Calls:** `LerpDrawVert`, `Transpose`, `InvertErrorTable`, `InvertCtrl`, `PutPointsOnCurve`, `MakeMeshNormals`, `R_CreateSurfaceGridMesh`, `VectorSubtract`, `VectorNormalize`, `DotProduct`, `VectorScale`, `VectorLengthSquared`, `sqrt`.
- **Notes:** Subdivision loop processes columns in each direction, backs up (`j -= 2`) if a column was split so it will be re-evaluated. Error value `999` marks colinear columns for removal. Width grows by 2 per split; guarded against `MAX_GRID_SIZE`.

### R_GridInsertColumn / R_GridInsertRow
- **Signature:** `srfGridMesh_t *R_GridInsertColumn(srfGridMesh_t *grid, int column, int row, vec3_t point, float loderror)` / `…InsertRow(…)`
- **Purpose:** Inserts a new column (or row) into an existing grid mesh at the given index, optionally overriding one vertex position. Used for patch stitching to align LOD boundaries between adjacent patches.
- **Inputs:** Existing grid, insertion index, optional override position and LOD error value.
- **Outputs/Return:** New `srfGridMesh_t *`; returns `NULL` if grid would exceed `MAX_GRID_SIZE`.
- **Side effects:** Calls `R_FreeSurfaceGridMesh` on the input grid; allocates a replacement. Preserves `lodOrigin`/`lodRadius` from original.
- **Calls:** `LerpDrawVert`, `VectorCopy`, `MakeMeshNormals`, `R_FreeSurfaceGridMesh`, `R_CreateSurfaceGridMesh`.

## Control Flow Notes
This file participates in the **load/registration** phase, not the per-frame render loop. `R_SubdividePatchToGrid` is called from `tr_bsp.c` during BSP surface loading to pre-tessellate patch surfaces into `srfGridMesh_t` entries that the back end renders via `rb_surfaceTable[SF_GRID]`. `R_GridInsertColumn`/`R_GridInsertRow` are called during the same load phase for patch LOD stitching (controlled by `PATCH_STITCHING` define). No per-frame calls originate here.

## External Dependencies
- **`tr_local.h`** — pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`
- **`ri`** (`refimport_t`) — `ri.Malloc`, `ri.Free`, `ri.Hunk_Alloc` for memory management
- **`r_subdivisions`** (`cvar_t *`) — controls adaptive subdivision error threshold
- **`MAX_GRID_SIZE` (65), `MAX_PATCH_SIZE` (32)** — defined in `tr_local.h`
- **`srfGridMesh_t`, `drawVert_t`, `SF_GRID`** — defined in `tr_local.h` / `qfiles.h`
- **Vector math macros** (`VectorSubtract`, `CrossProduct`, etc.) — defined in `q_shared.h`
- **`Com_Memcpy`, `Com_Memset`** — defined elsewhere in qcommon
