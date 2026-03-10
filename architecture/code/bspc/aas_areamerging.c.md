# code/bspc/aas_areamerging.c

## File Purpose
Implements the area-merging pass of the BSPC (BSP Compiler) AAS (Area Awareness System) generation pipeline. It iterates over temporary AAS areas, tests adjacent area pairs for convexity compatibility, and merges qualifying pairs into a single new convex area to reduce the total area count.

## Core Responsibilities
- Test whether two faces from different areas would form a non-convex region if merged (`NonConvex`)
- Validate merge eligibility: matching presence type, contents, and model number
- Detect ground/gap face flag conflicts that would block a merge
- Construct a new merged `tmp_area_t` by adopting all non-separating faces from both source areas
- Mark source areas as invalid and point them to the merged area via `mergedarea`
- Drive a two-phase merge loop: grounded areas first, then all areas, until no further merges occur
- Refresh the BSP tree's leaf pointers to follow `mergedarea` chains after merging

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tmp_area_t` | struct (typedef) | Temporary AAS area holding face list, presence/content/model info, invalid flag, and `mergedarea` forward pointer |
| `tmp_face_t` | struct (typedef) | Temporary AAS face with winding, plane number, front/back area pointers, face flags, and doubly-linked per-area chains |
| `tmp_node_t` | struct (typedef) | Temporary BSP tree node; leaf nodes reference a `tmp_area_t` |
| `plane_t` | struct (typedef) | Map plane with normal and distance; used for convexity dot-product tests |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tmpaasworld` | `tmp_aas_t` | global (defined in `aas_create.c`) | The entire in-progress temporary AAS world; iterated for area merging |
| `mapplanes` | `plane_t[]` | global (defined in `map.c`) | Array of all map planes; indexed by `face->planenum ^ side` |

## Key Functions / Methods

### AAS_RefreshMergedTree_r
- **Signature:** `tmp_node_t *AAS_RefreshMergedTree_r(tmp_node_t *tmpnode)`
- **Purpose:** Post-merge tree fixup — walks the BSP node tree recursively and updates each area leaf's `tmparea` pointer to the final merged area by following `mergedarea` chains.
- **Inputs:** `tmpnode` — current BSP node to process (NULL = solid leaf)
- **Outputs/Return:** Returns the (possibly updated) node pointer
- **Side effects:** Mutates `tmpnode->tmparea` in-place for area leaves
- **Calls:** Itself recursively on `children[0]` and `children[1]`
- **Notes:** Terminates on NULL (solid leaf). Chain-following loop handles multi-hop merges (A→B→C).

---

### NonConvex
- **Signature:** `int NonConvex(tmp_face_t *face1, tmp_face_t *face2, int side1, int side2)`
- **Purpose:** Convexity guard — returns true if any vertex of either face lies behind the plane of the other face beyond `CONVEX_EPSILON`, indicating a merge would produce a non-convex volume.
- **Inputs:** Two faces and their respective sides relative to their parent areas
- **Outputs/Return:** `int` — non-zero if non-convex, 0 if safe to merge
- **Side effects:** None
- **Calls:** `DotProduct` (macro)
- **Notes:** Uses `face->planenum ^ side` to select the correct plane orientation. `CONVEX_EPSILON = 0.3` provides a small tolerance.

---

### AAS_TryMergeFaceAreas
- **Signature:** `int AAS_TryMergeFaceAreas(tmp_face_t *seperatingface)`
- **Purpose:** Core merge operation — given the shared face between two areas, checks all eligibility conditions and, if satisfied, creates a new merged area and retires the two originals.
- **Inputs:** `seperatingface` — the `tmp_face_t` whose `frontarea`/`backarea` are the merge candidates
- **Outputs/Return:** `int` — true (1) if merge succeeded, false (0) otherwise
- **Side effects:** Allocates a new `tmp_area_t` via `AAS_AllocTmpArea`; calls `AAS_RemoveFaceFromArea`, `AAS_AddFaceSideToArea`, `AAS_FreeTmpFace`; marks both source areas `invalid = true` and sets their `mergedarea`; calls `AAS_CheckArea` and `AAS_FlipAreaFaces` on the new area
- **Calls:** `AAS_GapFace`, `NonConvex`, `AAS_AllocTmpArea`, `AAS_RemoveFaceFromArea`, `AAS_AddFaceSideToArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`, `AAS_FlipAreaFaces`, `Error`
- **Notes:** Skips separating faces (shared between the two areas) when migrating faces to the new area. Ground+gap conflict check prevents merging areas where one has gap faces and the other has ground faces.

---

### AAS_GroundArea
- **Signature:** `int AAS_GroundArea(tmp_area_t *tmparea)`
- **Purpose:** Returns true if any face in the area carries the `FACE_GROUND` flag.
- **Inputs:** `tmparea` — area to inspect
- **Outputs/Return:** `int` — 1 if grounded, 0 otherwise
- **Side effects:** None
- **Calls:** None directly (iterates `tmparea->tmpfaces`)
- **Notes:** Used as a filter in `AAS_MergeAreas` to prioritize ground-to-ground merges.

---

### AAS_MergeAreas
- **Signature:** `void AAS_MergeAreas(void)`
- **Purpose:** Top-level merge driver — iterates all valid areas, attempts to merge each adjacent area pair, and repeats until no further merges occur. Runs grounded areas first, then all areas.
- **Inputs:** None (operates on global `tmpaasworld`)
- **Outputs/Return:** void
- **Side effects:** Modifies `tmpaasworld.areas` list (areas invalidated, new areas added); calls `AAS_RefreshMergedTree_r` on `tmpaasworld.nodes`; writes progress via `Log_Write` and `qprintf`
- **Calls:** `AAS_GroundArea`, `AAS_TryMergeFaceAreas`, `AAS_RefreshMergedTree_r`, `Log_Write`, `qprintf`
- **Notes:** The outer `while(1)` loop alternates `groundfirst=true` (grounded-only pass) then `groundfirst=false` (all areas), breaking only when both passes produce zero merges. A commented-out earlier implementation is preserved in the file.

## Control Flow Notes
This file is a **build-time tool** (BSPC), not part of the game runtime. It executes during AAS file generation: after initial area creation (`aas_create.c`) and before edge/face optimization and file storage (`aas_store.c`). `AAS_MergeAreas` is called once per map conversion. No frame/update loop is involved.

## External Dependencies
- `qbsp.h` — `mapplanes`, `plane_t`, `winding_t`, `DotProduct`, `Error`, `qprintf`
- `aasfile.h` — `FACE_GROUND`, `FACE_GAP` flag constants
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmp_aas_t`, `tmpaasworld`; functions `AAS_AllocTmpArea`, `AAS_RemoveFaceFromArea`, `AAS_AddFaceSideToArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`, `AAS_FlipAreaFaces`, `AAS_GapFace`
- `aas_store.h` — included transitively via `aas_create.h`; `aasworld` global
- `Log_Write` — defined elsewhere (logging utility from `l_log.c`)
