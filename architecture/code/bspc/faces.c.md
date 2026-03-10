# code/bspc/faces.c

## File Purpose
Implements BSP face construction for the BSPC tool, handling vertex deduplication, T-junction elimination, face merging, and face subdivision. It transforms raw portal windings into properly split and merged BSP faces ready for BSP file output.

## Core Responsibilities
- Deduplicate map vertices via spatial hashing or linear search (`GetVertexnum`)
- Emit and assign vertex indices to face windings (`EmitFaceVertexes`, `EmitVertexes_r`)
- Detect and fix T-junctions by splitting edges at intermediate vertices (`TestEdge`, `FixFaceEdges`, `FixEdges_r`)
- Merge coplanar, same-content, same-texinfo faces on the same node (`TryMerge`, `MergeNodeFaces`)
- Subdivide faces exceeding the surface-cache size limit (`SubdivideFace`, `SubdivideNodeFaces`)
- Allocate/free `face_t` objects and build them from BSP portals (`FaceFromPortal`)
- Track and emit shared BSP edges, preventing four-way edges (`GetEdge2`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `face_t` | struct (from `qbsp.h`) | BSP face: winding, plane, texinfo, merge/split chain, vertex indices |
| `node_t` | struct (from `qbsp.h`) | BSP tree node/leaf; holds face list and portal list |
| `portal_t` | struct (from `qbsp.h`) | Portal between two nodes; source for visible faces |
| `hashvert_t` | struct | Linked-list node for the spatial vertex hash table (declared but unused; hash is done via `vertexchain[]` arrays) |
| `dedge_t` | struct (external) | BSP output edge: two vertex indices |
| `dvertex_t` | struct (external) | BSP output vertex: 3D point |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `c_merge`, `c_subdivide`, `c_nodefaces` | `int` | global | Counters for merged/subdivided/created faces |
| `c_totalverts`, `c_uniqueverts`, `c_degenerate`, `c_tjunctions`, `c_faceoverflows`, `c_facecollapse`, `c_badstartverts` | `int` | global | Diagnostic counters for vertex/edge processing |
| `superverts[MAX_SUPERVERTS]`, `numsuperverts` | `int[]`, `int` | global | Scratch buffer accumulating vertex indices during T-junction fixing |
| `edgefaces[MAX_MAP_EDGES][2]` | `face_t*[][]` | global | Maps each BSP edge to the (up to) two faces sharing it |
| `firstmodeledge`, `firstmodelface` | `int` | global | Offsets into the BSP edge/face arrays for the current model |
| `vertexchain[MAX_MAP_VERTS]` | `int[]` | global | Linked-list chains within each hash bucket for vertex dedup |
| `hashverts[HASH_SIZE*HASH_SIZE]` | `int[]` | global | Spatial hash table: maps bucket → first vertex index |
| `edge_dir`, `edge_start`, `edge_len` | `vec3_t`, `vec_t` | global | Current edge geometry used during recursive `TestEdge` |
| `edge_verts[MAX_MAP_VERTS]`, `num_edge_verts` | `int[]`, `int` | global | Candidate vertices to test against the current edge |
| `c_faces` | `int` | global | Live `face_t` allocation counter |

## Key Functions / Methods

### HashVec
- **Signature:** `unsigned HashVec(vec3_t vec)`
- **Purpose:** Maps a 3D point to a 2D hash bucket index using X/Y coordinates.
- **Inputs:** `vec` — world-space point
- **Outputs/Return:** Bucket index in `[0, HASH_SIZE²)`
- **Side effects:** Calls `Error` if point is outside ±4096 range
- **Calls:** `Error`
- **Notes:** Only uses X and Y; Z is ignored — assumes map geometry is spread in XY.

### GetVertexnum (hashed variant)
- **Signature:** `int GetVertexnum(vec3_t in)`
- **Purpose:** Finds or creates a deduplicated vertex in `dvertexes[]`, snapping near-integer values.
- **Inputs:** `in` — candidate vertex position
- **Outputs/Return:** Index into `dvertexes[]`
- **Side effects:** May append to `dvertexes[]`, `vertexchain[]`, `hashverts[]`; increments `numvertexes`, `c_totalverts`, `c_uniqueverts`
- **Calls:** `HashVec`, `Q_rint`, `Error`
- **Notes:** Linear fallback variant exists under `#else`; both share the same signature.

### FaceFromSuperverts
- **Signature:** `void FaceFromSuperverts(node_t *node, face_t *f, int base)`
- **Purpose:** Copies vertex indices from `superverts[]` into a face, splitting into a chain of faces if vertex count exceeds `MAXEDGES`.
- **Side effects:** Allocates new `face_t` nodes via `NewFaceFromFace`; inserts into `node->faces`; increments `c_faceoverflows`
- **Calls:** `NewFaceFromFace`

### EmitFaceVertexes
- **Signature:** `void EmitFaceVertexes(node_t *node, face_t *f)`
- **Purpose:** Converts a face winding's points to vertex indices (via `GetVertexnum` or raw allocation), then calls `FaceFromSuperverts`.
- **Side effects:** Writes to `superverts[]`, `dvertexes[]`, global counters
- **Calls:** `GetVertexnum`, `FaceFromSuperverts`

### FixTjuncs
- **Signature:** `void FixTjuncs(node_t *headnode)`
- **Purpose:** Top-level entry: first emits/deduplicates all vertices, then fixes T-junctions across the entire BSP tree.
- **Side effects:** Resets `hashverts[]`, all counters; calls recursive passes; prints stats via `qprintf`
- **Calls:** `EmitVertexes_r`, `FixEdges_r`, `qprintf`

### TestEdge
- **Signature:** `void TestEdge(vec_t start, vec_t end, int p1, int p2, int startvert)`
- **Purpose:** Recursively detects intermediate vertices lying on the edge p1→p2 and breaks the edge at each T-junction, appending results to `superverts[]`.
- **Side effects:** Increments `c_tjunctions`, `c_degenerate`, `numsuperverts`; recurses
- **Calls:** `VectorCopy`, `VectorSubtract`, `DotProduct`, `VectorMA`, `VectorLength`, `Error`
- **Notes:** Reentrancy is intentional; uses global `edge_start`/`edge_dir`.

### GetEdge2
- **Signature:** `int GetEdge2(int v1, int v2, face_t *f)`
- **Purpose:** Finds an existing reverse edge for sharing, or emits a new `dedge_t`.
- **Outputs/Return:** Positive index for a new/forward edge; negative index for a shared reverse edge
- **Side effects:** Writes `dedges[]`, `edgefaces[]`; increments `numedges`, `c_tryedges`
- **Calls:** `Error`
- **Notes:** Prevents four-way edge sharing; controlled by `noshare` flag.

### TryMerge
- **Signature:** `face_t *TryMerge(face_t *f1, face_t *f2, vec3_t planenormal)`
- **Purpose:** Attempts to merge two coplanar faces sharing an edge into one; returns the merged face or NULL.
- **Side effects:** Sets `f1->merged`, `f2->merged`; allocates a new face; increments `c_merge`
- **Calls:** `TryMergeWinding`, `NewFaceFromFace`

### SubdivideFace
- **Signature:** `void SubdivideFace(node_t *node, face_t *f)`
- **Purpose:** Recursively chops a face along texture axes until all pieces fit within `subdivide_size`. Skips warped/sky surfaces.
- **Side effects:** Allocates split faces; inserts into `node->faces`; increments `c_subdivide`
- **Calls:** `ClipWindingEpsilon`, `NewFaceFromFace`, `VectorNormalize`, `DotProduct`

### MakeFaces / MakeFaces_r
- **Signature:** `void MakeFaces(node_t *node)` / `void MakeFaces_r(node_t *node)`
- **Purpose:** Entry point: recursively walks the BSP tree, creates faces from valid portals, then merges and subdivides at each internal node.
- **Side effects:** Allocates `face_t` via `FaceFromPortal`; calls `MergeNodeFaces`, `SubdivideNodeFaces`; increments `c_nodefaces`
- **Calls:** `FaceFromPortal`, `MergeNodeFaces`, `SubdivideNodeFaces`, `qprintf`

## Control Flow Notes
Called during the BSP compilation pipeline after portalization. Typical call order:
1. `MakeFaces(headnode)` — creates faces from portals, merges, subdivides
2. `FixTjuncs(headnode)` — snaps vertices and fixes T-junctions
3. `GetEdge2(...)` — called per-face by `writebsp.c` to emit final edge list

## External Dependencies
- **Includes:** `qbsp.h` (all BSP types, flags, map globals), `l_mem.h` (GetMemory/FreeMemory)
- **Defined elsewhere:** `dvertexes[]`, `numvertexes`, `dedges[]`, `numedges`, `texinfo[]`, `noweld`, `notjunc`, `nomerge`, `nosubdiv`, `noshare`, `subdivide_size` (from `bspc.c`/map globals); `TryMergeWinding`, `CopyWinding`, `ReverseWinding`, `FreeWinding`, `ClipWindingEpsilon` (from winding/poly library); `qprintf`, `Error` (from utility layer)
