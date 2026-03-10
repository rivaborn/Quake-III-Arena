# code/renderer/tr_marks.c

## File Purpose
Implements polygon projection ("marks") onto world geometry for decal-like effects such as bullet holes and scorch marks. It traverses the BSP tree to collect candidate surfaces, clips projected polygons against those surfaces, and returns fragments for use by the cgame.

## Core Responsibilities
- Traverse the BSP tree to collect surfaces within an AABB (`R_BoxSurfaces_r`)
- Clip a polygon against a half-space plane (`R_ChopPolyBehindPlane`)
- Clip surface triangles against the projection volume's bounding planes (`R_AddMarkFragments`)
- Project a mark polygon onto planar (`SF_FACE`) and curved grid (`SF_GRID`) world surfaces (`R_MarkFragments`)
- Filter surfaces by shader flags (`SURF_NOIMPACT`, `SURF_NOMARKS`, `CONTENTS_FOG`) and face angle relative to projection direction

## Key Types / Data Structures

None defined in this file; relies on types from `tr_local.h`.

| Name | Kind | Purpose |
|---|---|---|
| `markFragment_t` | struct (defined elsewhere) | Records one clipped polygon fragment: first point index and point count |
| `srfSurfaceFace_t` | struct (defined elsewhere) | Planar BSP face surface data including plane, indices, and vertex points |
| `srfGridMesh_t` | struct (defined elsewhere) | Bezier grid mesh with per-vertex positions and normals |
| `mnode_t` | struct (defined elsewhere) | BSP tree node/leaf used during surface collection |
| `msurface_t` | struct (defined elsewhere) | World surface with shader and surface data pointer |

## Global / File-Static State

None (reads `tr.viewCount`, `tr.world` from the global `trGlobals_t tr`).

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tr` | `trGlobals_t` | global (defined in `tr_init.c`) | Accessed for `tr.viewCount` and `tr.world->nodes` |

## Key Functions / Methods

### R_ChopPolyBehindPlane
- **Signature:** `static void R_ChopPolyBehindPlane(int numInPoints, vec3_t inPoints[MAX_VERTS_ON_POLY], int *numOutPoints, vec3_t outPoints[MAX_VERTS_ON_POLY], vec3_t normal, vec_t dist, vec_t epsilon)`
- **Purpose:** Sutherland-Hodgman single-plane polygon clip; discards geometry on the back side of the plane.
- **Inputs:** Input polygon vertices, plane normal + distance, epsilon tolerance.
- **Outputs/Return:** Writes clipped polygon into `outPoints`, sets `*numOutPoints`.
- **Side effects:** None.
- **Calls:** `DotProduct`, `VectorCopy`, `Com_Memcpy`.
- **Notes:** Aborts (outputs 0 points) if `numInPoints >= MAX_VERTS_ON_POLY - 2` to avoid overflow. Uses linear interpolation to generate split points. `outPoints` must have capacity for `numInPoints + 2`.

---

### R_BoxSurfaces_r
- **Signature:** `void R_BoxSurfaces_r(mnode_t *node, vec3_t mins, vec3_t maxs, surfaceType_t **list, int listsize, int *listlength, vec3_t dir)`
- **Purpose:** Recursively (loop-optimized for the tail case) descends the BSP tree and collects all surfaces whose leaves intersect the given AABB.
- **Inputs:** BSP node, bounding box, output surface list, projection direction `dir`.
- **Outputs/Return:** Appends `surfaceType_t*` pointers to `list`, increments `*listlength`.
- **Side effects:** Sets `surf->viewCount = tr.viewCount` to mark surfaces as visited/rejected.
- **Calls:** `BoxOnPlaneSide`, `DotProduct`, recursive `R_BoxSurfaces_r`.
- **Notes:** Surfaces with `SURF_NOIMPACT`, `SURF_NOMARKS`, or `CONTENTS_FOG` are rejected. For `SF_FACE`, additionally rejects faces making an angle shallower than ~60° with the projection direction (`DotProduct > -0.5`). Non-`SF_FACE`/non-`SF_GRID` surfaces are rejected. Uses `viewCount` to avoid adding a surface twice when it spans multiple leaves.

---

### R_AddMarkFragments
- **Signature:** `void R_AddMarkFragments(int numClipPoints, vec3_t clipPoints[2][MAX_VERTS_ON_POLY], int numPlanes, vec3_t *normals, float *dists, int maxPoints, vec3_t pointBuffer, int maxFragments, markFragment_t *fragmentBuffer, int *returnedPoints, int *returnedFragments, vec3_t mins, vec3_t maxs)`
- **Purpose:** Clips one candidate triangle/polygon through all bounding planes of the projection volume and, if any polygon remains, appends it as a `markFragment_t`.
- **Inputs:** Starting clip polygon, projection volume planes, output buffers.
- **Outputs/Return:** Appends points into `pointBuffer`, records a `markFragment_t` in `fragmentBuffer`.
- **Side effects:** Increments `*returnedPoints` and `*returnedFragments`.
- **Calls:** `R_ChopPolyBehindPlane`, `Com_Memcpy`.
- **Notes:** Ping-pong buffers (`clipPoints[0]` / `clipPoints[1]`) avoid extra allocation. Returns early if the clipped polygon is empty or output buffers are full.

---

### R_MarkFragments
- **Signature:** `int R_MarkFragments(int numPoints, const vec3_t *points, const vec3_t projection, int maxPoints, vec3_t pointBuffer, int maxFragments, markFragment_t *fragmentBuffer)`
- **Purpose:** Main entry point. Given a mark polygon and a projection vector, finds all world surface fragments that the polygon projects onto and returns clipped fragment geometry.
- **Inputs:** Mark polygon vertices, projection vector, output point/fragment buffers with capacities.
- **Outputs/Return:** Returns count of generated fragments; fills `pointBuffer` and `fragmentBuffer`.
- **Side effects:** Increments `tr.viewCount` (used as a stamp to deduplicate surfaces).
- **Calls:** `VectorNormalize2`, `ClearBounds`, `AddPointToBounds`, `VectorAdd`, `VectorMA`, `VectorSubtract`, `CrossProduct`, `VectorNormalizeFast`, `DotProduct`, `VectorCopy`, `VectorInverse`, `R_BoxSurfaces_r`, `R_AddMarkFragments`.
- **Notes:** Constructs `numPoints + 2` bounding planes (one per polygon edge plus near/far). Handles `SF_GRID` by manually triangulating each quad cell; handles `SF_FACE` via stored index list. `SF_TRIANGLES` and others are explicitly ignored. `MARKER_OFFSET` is 0 (the normal-offset path is compiled out).

## Control Flow Notes
Called from the cgame/renderer interface (`RE_MarkFragments` or equivalent), outside the main per-frame render path. Invoked on impact events (bullet hits, explosions). Increments `tr.viewCount` each call to reuse the surface deduplication stamp that the main view traversal also uses.

## External Dependencies
- **Includes:** `tr_local.h` (all renderer types, `trGlobals_t tr`, math macros)
- **Defined elsewhere:** `tr` (`trGlobals_t`), `BoxOnPlaneSide`, `DotProduct`, `CrossProduct`, `VectorNormalize2`, `VectorNormalizeFast`, `VectorMA`, `VectorAdd`, `VectorSubtract`, `VectorCopy`, `VectorInverse`, `ClearBounds`, `AddPointToBounds`, `Com_Memcpy`, `markFragment_t`, `srfSurfaceFace_t`, `srfGridMesh_t`, `drawVert_t`, `SURF_NOIMPACT`, `SURF_NOMARKS`, `CONTENTS_FOG`, `VERTEXSIZE`
