# code/bspc/aas_facemerging.c

## File Purpose
Implements face-merging passes over the temporary AAS world during BSP-to-AAS conversion. It reduces face count by coalescing coplanar, compatible faces within and across areas, simplifying the final AAS geometry.

## Core Responsibilities
- Attempt to merge two individual `tmp_face_t` windings into one (`AAS_TryMergeFaces`)
- Iterate all areas, retrying merges until no more are possible (`AAS_MergeAreaFaces`)
- Merge all same-plane faces within a single area unconditionally (`AAS_MergePlaneFaces`)
- Guard plane-face merges with a compatibility pre-check (`AAS_CanMergePlaneFaces`)
- Drive a full pass of per-plane face merging over all areas (`AAS_MergeAreaPlaneFaces`)
- Clean up consumed faces: remove from area lists and free them

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `tmp_face_t` | struct (typedef) | Temporary AAS face: holds winding, plane number, front/back area pointers, face flags, and doubly-linked list links |
| `tmp_area_t` | struct (typedef) | Temporary AAS area: linked list of faces, validity flag, area number |
| `tmp_aas_t` | struct (typedef) | Root temporary AAS world; `tmpaasworld` is the global instance |
| `winding_t` | struct (opaque) | Convex polygon winding used to represent face geometry |
| `plane_t` | struct | Map plane with normal and distance; stored in `mapplanes[]` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tmpaasworld` | `tmp_aas_t` | global (defined in `aas_create.c`) | Root temporary AAS world; provides the area linked list iterated by merge passes |
| `mapplanes` | `plane_t[]` | global (defined in `map.c`) | Array of all map planes; indexed by `tmp_face_t::planenum` to obtain plane normals |

## Key Functions / Methods

### AAS_TryMergeFaces
- **Signature:** `int AAS_TryMergeFaces(tmp_face_t *face1, tmp_face_t *face2)`
- **Purpose:** Attempts to merge `face2` into `face1`. On success, `face1->winding` is replaced with the merged winding and `face2` is freed.
- **Inputs:** Two candidate `tmp_face_t` pointers.
- **Outputs/Return:** `true` (1) if merge succeeded; `false` (0) otherwise.
- **Side effects:** Frees `face2->winding` and the face itself via `AAS_FreeTmpFace`; removes `face2` from its front/back areas; mutates `face1->winding`.
- **Calls:** `MergeWindings`, `TryMergeWinding`, `FreeWinding`, `AAS_RemoveFaceFromArea`, `AAS_FreeTmpFace`, `Log_Write`.
- **Notes:** Compatibility requires matching `faceflags`, `frontarea`, `backarea`, and `planenum`. If both areas are real (non-zero), `MergeWindings` is used (unconstrained hull merge); otherwise `TryMergeWinding` (strict edge-sharing merge). An alternate commented-out implementation exists showing a previous design that also accepted flipped front/back pairs.

### AAS_MergeAreaFaces
- **Signature:** `void AAS_MergeAreaFaces(void)`
- **Purpose:** Full-world pass: for each area, tries all face pairs; restarts the inner loop after any successful merge.
- **Inputs:** None (reads `tmpaasworld.areas`).
- **Outputs/Return:** None.
- **Side effects:** Modifies `tmpaasworld` area face lists; calls `AAS_CheckArea` after each merge; prints progress via `qprintf`/`Log_Write`.
- **Calls:** `AAS_TryMergeFaces`, `AAS_CheckArea`, `Log_Write`, `qprintf`.
- **Notes:** Uses `lasttmparea` to re-examine the previous area after a restart, ensuring no pair is skipped.

### AAS_MergePlaneFaces
- **Signature:** `void AAS_MergePlaneFaces(tmp_area_t *tmparea, int planenum)`
- **Purpose:** Unconditionally merges all faces of `tmparea` that lie in the given plane (or its flip) using `MergeWindings`.
- **Inputs:** Target area, plane number to match.
- **Outputs/Return:** None.
- **Side effects:** Frees consumed faces; mutates surviving face's winding; removes merged faces from their areas.
- **Calls:** `MergeWindings`, `FreeWinding`, `AAS_RemoveFaceFromArea`, `AAS_FreeTmpFace`.
- **Notes:** Does not check flag or area compatibility — caller (`AAS_MergeAreaPlaneFaces`) must gate with `AAS_CanMergePlaneFaces` first. Uses `nextface2` saved before inner-loop body to handle iterator invalidation.

### AAS_CanMergePlaneFaces
- **Signature:** `int AAS_CanMergePlaneFaces(tmp_area_t *tmparea, int planenum)`
- **Purpose:** Checks whether there are ≥2 coplanar faces in `tmparea` sharing the same front area, back area, and face flags — i.e., a merge would be valid.
- **Inputs:** Target area, plane number.
- **Outputs/Return:** `true` if a mergeable group exists; `false` otherwise.
- **Side effects:** None.
- **Calls:** None beyond struct field access.

### AAS_MergeAreaPlaneFaces
- **Signature:** `void AAS_MergeAreaPlaneFaces(void)`
- **Purpose:** Full-world pass: for each area, finds any plane with multiple mergeable faces and calls `AAS_MergePlaneFaces`, re-examining the same area afterward.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Modifies `tmpaasworld`; prints progress.
- **Calls:** `AAS_CanMergePlaneFaces`, `AAS_MergePlaneFaces`, `Log_Write`, `qprintf`.

## Control Flow Notes
This file is an offline map-compilation tool pass, not runtime game code. It runs during AAS generation inside `bspc`, after the BSP-to-AAS conversion produces a temporary world (`tmpaasworld`). `AAS_MergeAreaFaces` and `AAS_MergeAreaPlaneFaces` are called sequentially as geometry optimization steps before the final AAS file is written. There is no per-frame update loop.

## External Dependencies
- **`qbsp.h`** — pulls in `mapplanes[]`, `winding_t`, `plane_t`, `Log_Write`, `qprintf`, `MergeWindings`, `TryMergeWinding`, `FreeWinding`
- **`../botlib/aasfile.h`** — AAS file format constants (face flags, area flags, travel types); types not directly used in this file but needed by the broader AAS creation pipeline
- **`aas_create.h`** — `tmp_face_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld`, `AAS_RemoveFaceFromArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`
- **Defined elsewhere:** `tmpaasworld` (aas_create.c), `mapplanes` (map.c), `MergeWindings`/`TryMergeWinding`/`FreeWinding` (l_poly.c)
