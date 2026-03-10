# code/bspc/tetrahedron.c

## File Purpose
Implements a tetrahedral decomposition algorithm for the BSPC tool, converting an AAS (Area Awareness System) world's solid faces into a triangle mesh and then subdividing that mesh into tetrahedrons. This is a spatial decomposition utility used during AAS file processing.

## Core Responsibilities
- Allocate and free the global `thworld` data store for tetrahedron construction
- Manage hashed pools of vertices, planes, edges, triangles, and tetrahedrons with find-or-create semantics
- Validate candidate edges and triangles against existing geometry to prevent intersections
- Search for valid tetrahedrons using two strategies: shared-edge pairing (`TH_FindTetrahedron1`) and single-triangle + free vertex (`TH_FindTetrahedron2`)
- Drive the full decomposition loop (`TH_TetrahedralDecomposition`) until no new tetrahedrons can be formed
- Convert AAS solid faces into triangles (`TH_CreateAASFaceTriangles`, `TH_AASToTriangleMesh`)
- Provide the top-level entry point `TH_AASToTetrahedrons` that loads an AAS file and runs the full pipeline

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `th_plane_t` | struct | Plane with normal, dist, type, signbits, and hash chain |
| `th_vertex_t` | struct | 3D vertex with usage counter and hash chain |
| `th_edge_t` | struct | Directed edge (two vertex indices), usage counter, hash chain |
| `th_triangle_t` | struct | Triangle with 3 edge indices (signed for direction), bounding planes, plane number, front/back tetrahedron refs, bbox, and list/hash links |
| `th_tetrahedron_t` | struct | Four triangle indices (signed) and a volume field |
| `th_t` | struct | Monolithic world container: counts + arrays + hash tables for all above types |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `thworld` | `th_t` | global | Single global instance holding all tetrahedron world data during a build pass |

## Key Functions / Methods

### TH_InitMaxTH
- Signature: `void TH_InitMaxTH(void)`
- Purpose: Allocates zeroed memory for all five arrays in `thworld` and resets all hash tables.
- Inputs: None
- Outputs/Return: None
- Side effects: Heap allocations via `GetClearedMemory`; writes to `thworld`
- Calls: `GetClearedMemory`, `memset`

### TH_FreeMaxTH
- Signature: `void TH_FreeMaxTH(void)`
- Purpose: Frees all arrays in `thworld` and zeroes counts/pointers.
- Inputs: None
- Outputs/Return: None
- Side effects: Heap frees; zeroes `thworld` fields

### TH_FindFloatPlane / TH_CreateFloatPlane
- Signature: `int TH_FindFloatPlane(vec3_t normal, vec_t dist)` / `int TH_CreateFloatPlane(vec3_t normal, vec_t dist)`
- Purpose: Hash-based plane deduplication. Snaps near-axial normals/distances, searches ±1 hash bins, creates a plane pair (plane + flipped) if not found.
- Inputs: Normal vector, distance
- Outputs/Return: Index into `thworld.planes`
- Side effects: May append two entries to `thworld.planes`; updates `thworld.planehash`
- Notes: Axial planes are stored with positive-facing normal first.

### TH_FindOrCreateVertex / TH_FindOrCreateEdge / TH_FindOrCreateTriangle (via TH_CreateTriangle)
- Shared pattern across geometry types: hash lookup → create on miss, return index.
- Edge indices are signed: negative means the stored edge is traversed in reverse.

### TH_TryEdge
- Signature: `int TH_TryEdge(int v1, int v2)`
- Purpose: Validates a candidate edge by checking it does not intersect any existing open triangle (triangles not yet enclosed by two tetrahedrons).
- Inputs: Two vertex indices
- Outputs/Return: `true` if edge is valid or already exists
- Calls: `TH_FindEdge`, `TH_OutsideBoundingBox`, `TH_IntersectTrianglePlanes`

### TH_TryTriangle
- Signature: `int TH_TryTriangle(int verts[3])`
- Purpose: Validates a candidate triangle by checking no existing open edge pierces it.
- Inputs: Three vertex indices
- Outputs/Return: `true` if triangle placement is valid
- Calls: `TH_CreateTrianglePlanes`, `TH_IntersectTrianglePlanes`

### TH_FindTetrahedron1
- Signature: `int TH_FindTetrahedron1(th_triangle_t *tri, int *triangles)`
- Purpose: Attempts to form a tetrahedron by finding a second triangle sharing an edge with `tri`, then validating/creating the two required side triangles.
- Inputs: Seed triangle pointer, output array for 4 triangle indices
- Outputs/Return: `true` on success; fills `triangles[4]`
- Calls: `TH_TryEdge`, `TH_TryTriangle`, `TH_FindTriangle`, `TH_CreateTriangle`

### TH_FindTetrahedron2
- Signature: `int TH_FindTetrahedron2(th_triangle_t *tri, int *triangles)`
- Purpose: Fallback — iterates all vertices to find one not coplanar with `tri` from which valid edges and three new side triangles can be constructed.
- Inputs: Seed triangle, output array for 4 triangle indices
- Outputs/Return: `true` on success

### TH_TetrahedralDecomposition
- Signature: `void TH_TetrahedralDecomposition(th_triangle_t *triangles)`
- Purpose: Main decomposition loop. Iterates all open triangles, attempts `TH_FindTetrahedron1` (then `TH_FindTetrahedron2`) until no new tetrahedrons can be formed. Reports triangles still missing tetrahedrons.
- Side effects: Populates `thworld.tetrahedrons`; prints progress via `qprintf`/`Log_Print`
- Notes: Contains a large commented-out alternate list-based approach.

### TH_CreateAASFaceTriangles
- Signature: `th_triangle_t *TH_CreateAASFaceTriangles(aas_face_t *face)`
- Purpose: Triangulates a convex AAS face. Uses fan triangulation from a vertex if all triples are colinear, otherwise uses standard pivot triangulation. Marks all created triangles with `front = -1` (exterior).
- Calls: `TH_AASFaceVertex`, `TH_Colinear`, `TH_FaceCenter`, `TH_FindOrCreateVertex`, `TH_CreateTriangle`, `TH_AddTriangleToList`

### TH_AASToTetrahedrons
- Signature: `void TH_AASToTetrahedrons(char *filename)`
- Purpose: Top-level entry point. Loads AAS file, initializes world, builds triangle mesh from solid faces, runs decomposition, frees world.
- Calls: `AAS_LoadAASFile`, `TH_InitMaxTH`, `TH_AASToTriangleMesh`, `TH_TetrahedralDecomposition`, `TH_FreeMaxTH`

## Control Flow Notes
This file is a standalone offline tool pass, not part of the game runtime. It is invoked from BSPC (`bspc.c`) as a post-processing step on an already-compiled AAS file. There is no frame/update loop; execution is entirely synchronous from `TH_AASToTetrahedrons`.

## External Dependencies
- `qbsp.h` — BSP types, math macros (`DotProduct`, `CrossProduct`, `VectorNormalize`, etc.), `Error`, `qprintf`, `Log_Print`
- `l_mem.h` — `GetClearedMemory`, `FreeMemory`
- `botlib/aasfile.h` — `aas_face_t`, `aas_edge_t`, `FACE_SOLID`, AAS data structure definitions
- `aas_store.h` — `aasworld` global (type `aas_t`), AAS area/face/edge/vertex arrays
- `aas_cfg.h` — configuration types (included transitively, not directly used here)
- `aas_file.h` — `AAS_LoadAASFile` (defined elsewhere)
- `aasworld` — global AAS world state (defined in `be_aas_def` / `aas_store`)
