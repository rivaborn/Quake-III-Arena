# code/bspc/aas_create.h

## File Purpose
Defines the temporary in-memory data structures used during AAS (Area Awareness System) world construction, along with the public interface for creating and manipulating those structures. It serves as the shared type contract between the BSPC tool's BSP-to-AAS conversion pipeline stages.

## Core Responsibilities
- Declare the `tmp_face_t`, `tmp_area_t`, `tmp_areasettings_t`, `tmp_node_t`, `tmp_nodebuf_t`, and `tmp_aas_t` intermediate structs used during AAS compilation
- Expose the top-level `AAS_Create` entry point for converting a loaded BSP map into an `.AAS` file
- Provide allocator/free declarations for all temporary AAS primitives
- Declare face-to-area linkage and removal helpers
- Declare geometry query helpers (`AAS_GapFace`, `AAS_GroundFace`, `AAS_FlipAreaFaces`, `AAS_CheckArea`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `tmp_face_t` | struct | Temporary AAS face: holds plane, winding, front/back area pointers, flags, and doubly-linked list links per side |
| `tmp_areasettings_t` | struct | Metadata for a temporary area: contents, BSP model number, flags, presence type, reachability info |
| `tmp_area_t` | struct | Temporary AAS area: face list, presence type, contents, validity flag, merge pointer, and list links |
| `tmp_node_t` | struct | Temporary BSP node used during AAS tree construction: plane, optional area pointer, two children |
| `tmp_nodebuf_t` | struct | Fixed-size slab allocator block (`NODEBUF_SIZE=128`) for `tmp_node_t`, chained via `next` pointer |
| `tmp_aas_t` | struct | Root container for the entire temporary AAS world: face/area/node arrays, counts, and the node buffer chain |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tmpaasworld` | `tmp_aas_t` | global (extern) | Singleton holding all temporary AAS data during the BSP-to-AAS conversion pass |

## Key Functions / Methods

### AAS_Create
- **Signature:** `void AAS_Create(char *aasfile)`
- **Purpose:** Top-level entry point; converts an already-loaded BSP map into a compiled `.AAS` file written to `aasfile`.
- **Inputs:** `aasfile` — output file path string
- **Outputs/Return:** None (void); produces a file on disk
- **Side effects:** Reads global BSP world state; populates and then serializes `tmpaasworld`; file I/O
- **Calls:** Not inferable from this file (defined in `aas_create.c`)
- **Notes:** Caller must have loaded the BSP map before invoking

### AAS_AddFaceSideToArea
- **Signature:** `void AAS_AddFaceSideToArea(tmp_face_t *tmpface, int side, tmp_area_t *tmparea)`
- **Purpose:** Links one side of a temporary face into an area's face list, updating the doubly-linked `prev[side]`/`next[side]` pointers and the face's `frontarea`/`backarea` pointer.
- **Inputs:** `tmpface` — face to add; `side` — 0=front, 1=back; `tmparea` — target area
- **Outputs/Return:** None
- **Side effects:** Mutates `tmpface` and `tmparea` linked-list state
- **Calls:** Not inferable from this file
- **Notes:** `side` indexes into `tmp_face_t::prev[2]`/`next[2]`

### AAS_RemoveFaceFromArea
- **Signature:** `void AAS_RemoveFaceFromArea(tmp_face_t *tmpface, tmp_area_t *tmparea)`
- **Purpose:** Unlinks a face from the given area's face list.
- **Inputs:** `tmpface`, `tmparea`
- **Outputs/Return:** None
- **Side effects:** Mutates linked-list pointers in both structs
- **Calls:** Not inferable from this file

### AAS_GapFace / AAS_GroundFace
- **Signature:** `int AAS_GapFace(tmp_face_t *tmpface, int side)` / `int AAS_GroundFace(tmp_face_t *tmpface)`
- **Purpose:** Geometry classifiers; return non-zero if the face qualifies as a gap (a drop-off from the given side) or a walkable ground surface, respectively.
- **Inputs:** `tmpface`; `side` for gap check
- **Outputs/Return:** Boolean int
- **Side effects:** None (query only)
- **Notes:** Used during reachability analysis and area classification

### AAS_FlipAreaFaces / AAS_CheckArea
- **Signature:** `void AAS_FlipAreaFaces(tmp_area_t *)` / `void AAS_CheckArea(tmp_area_t *)`
- **Purpose:** `FlipAreaFaces` corrects face winding orientation so normals point outward from the area; `CheckArea` validates the area's geometric consistency.
- **Side effects:** `FlipAreaFaces` mutates windings; `CheckArea` may emit warnings/errors.

### Allocator/Free pairs
- `AAS_AllocTmpFace` / `AAS_FreeTmpFace`, `AAS_AllocTmpArea` / `AAS_FreeTmpArea`, `AAS_AllocTmpNode` / `AAS_FreeTmpNode`: Standard pool-style alloc/free for each temporary primitive type. Nodes are carved from `tmp_nodebuf_t` slabs chained in `tmpaasworld.nodebuffer`.

## Control Flow Notes
This header is consumed exclusively by the BSPC offline compiler tool (not the runtime engine). `AAS_Create` is called once per map compile after BSP loading. Internally it drives a pipeline: BSP tree traversal → temporary area/face construction → area merging (`aas_areamerging`) → reachability computation → serialization to `.AAS`.

## External Dependencies
- `winding_t` — polygon winding type; defined in `l_poly.h` / `qbsp.h` (not in this file)
- `AREA_PORTAL` (`1`) — flag constant used by face/area classification logic elsewhere in `bspc/`
- `NODEBUF_SIZE` (`128`) — compile-time slab size for node buffer
