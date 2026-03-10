# code/bspc/aas_edgemelting.c

## File Purpose
Implements the "edge melting" pass for AAS (Area Awareness System) world generation, which refines face winding geometry by inserting shared boundary vertices between adjacent faces within the same area. This is a preprocessing step in BSP-to-AAS conversion that improves topological accuracy of convex area boundaries.

## Core Responsibilities
- For each pair of faces in an AAS area, detect vertices of one face that lie on the boundary edge of another face's winding
- Insert those detected vertices into the target winding via `AddWindingPoint`, splitting edges where needed
- Accumulate and report a count of total winding edge splits across all areas
- Log progress to both console (`qprintf`) and log file (`Log_Write`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `tmp_face_t` | struct (typedef) | Temporary AAS face with winding, plane ref, and area linkage |
| `tmp_area_t` | struct (typedef) | Temporary AAS area owning a linked list of `tmp_face_t` |
| `winding_t` | struct (typedef, defined elsewhere) | Polygon winding — array of 3D points |
| `plane_t` | struct (typedef) | Map plane with normal, dist, type, signbits |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tmpaasworld` | `tmp_aas_t` | global (extern, defined in aas_create.c) | The entire in-progress temporary AAS world; iterated for all areas |
| `mapplanes` | `plane_t[]` | global (extern, defined in map.c) | Array of all BSP map planes; indexed by `face->planenum` |

## Key Functions / Methods

### AAS_MeltFaceWinding
- **Signature:** `int AAS_MeltFaceWinding(tmp_face_t *face1, tmp_face_t *face2)`
- **Purpose:** Attempts to insert vertices from `face2`'s winding into `face1`'s winding wherever a vertex of `face2` lies on an edge of `face1`. Marked `// FIXME: this is buggy`.
- **Inputs:** Two distinct `tmp_face_t` pointers; `face1` is the target to be modified, `face2` is the source of candidate vertices.
- **Outputs/Return:** Count of winding point insertions (splits) performed into `face1`.
- **Side effects:** Frees and replaces `face1->winding` for each inserted point (heap alloc/free via `AddWindingPoint` / `FreeWinding`).
- **Calls:** `PointOnWinding`, `AddWindingPoint`, `FreeWinding`; `Error` (debug only)
- **Notes:** The `#ifdef DEBUG` guard validates winding pointers. The in-loop reassignment `face1->winding = neww` after `FreeWinding` means subsequent iterations see the growing winding — order-dependent behavior. Known buggy per comment.

### AAS_MeltFaceWindingsOfArea
- **Signature:** `int AAS_MeltFaceWindingsOfArea(tmp_area_t *tmparea)`
- **Purpose:** Runs an O(N²) all-pairs melt pass over all faces of a single area, calling `AAS_MeltFaceWinding` for every ordered pair of distinct faces.
- **Inputs:** `tmp_area_t *tmparea` — a single temporary area whose `tmpfaces` linked list is iterated.
- **Outputs/Return:** Total split count for the area.
- **Side effects:** Mutates face windings in-place via `AAS_MeltFaceWinding`.
- **Calls:** `AAS_MeltFaceWinding`
- **Notes:** Side tracking (`side1`, `side2`) uses the face's `next[side]` linked list pointer to traverse both front/back face lists correctly. Skips self-pairs via `if (face1 == face2) continue`.

### AAS_MeltAreaFaceWindings
- **Signature:** `void AAS_MeltAreaFaceWindings(void)`
- **Purpose:** Top-level entry point; iterates all areas in `tmpaasworld` and calls `AAS_MeltFaceWindingsOfArea` on each, accumulating and reporting progress.
- **Inputs:** None (operates on `tmpaasworld` global).
- **Outputs/Return:** void.
- **Side effects:** Console progress output (`qprintf`), log file writes (`Log_Write`). Mutates face windings across the entire AAS world.
- **Calls:** `AAS_MeltFaceWindingsOfArea`, `qprintf`, `Log_Write`

## Control Flow Notes
This file is called during the **AAS build/compile phase** (offline tool `bspc`), not at game runtime. It sits in a post-BSP-conversion pipeline step after temporary areas and faces have been created (`AAS_Create`), and before area merging and final AAS file output. `AAS_MeltAreaFaceWindings` is the entry point invoked by the higher-level `AAS_Create` pipeline.

## External Dependencies
- `qbsp.h` — `plane_t`, `mapplanes[]`, `winding_t`, `qprintf`, logging utilities
- `../botlib/aasfile.h` — AAS data structure constants and types
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld`
- **Defined elsewhere:** `PointOnWinding`, `AddWindingPoint`, `FreeWinding` (winding geometry utilities, likely `l_poly.c`); `Log_Write` (`l_log.c`); `Error` (`l_cmd.c`)
