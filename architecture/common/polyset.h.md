# common/polyset.h

## File Purpose
Defines data structures and function declarations for managing collections of triangulated polygon sets used in model/geometry processing tools. It serves as a shared header for offline tools (map compiler, model exporters) rather than the runtime engine.

## Core Responsibilities
- Define compile-time limits for triangle and polyset counts
- Declare the `triangle_t` primitive (geometry + normals + UVs per tri)
- Declare the `polyset_t` container grouping named triangles with a material
- Expose the polyset utility API (load, collapse, split, snap, normal computation)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `st_t` | typedef (float[2]) | UV / texture coordinate pair per vertex |
| `rgb_t` | typedef (float[3]) | RGB color triple (declared but not used in any struct here) |
| `triangle_t` | struct | Single triangle: 3 positions, 3 normals, 3 UV coords |
| `polyset_t` | struct | Named group of triangles with an associated material name |

## Global / File-Static State
None.

## Key Functions / Methods

### Polyset_LoadSets
- **Signature:** `polyset_t *Polyset_LoadSets( const char *file, int *numpolysets, int maxTrisPerSet )`
- **Purpose:** Load one or more polysets from a file on disk.
- **Inputs:** `file` — path to source file; `numpolysets` — out-param for count; `maxTrisPerSet` — triangle budget per set
- **Outputs/Return:** Pointer to allocated array of `polyset_t`; `*numpolysets` written on success
- **Side effects:** Allocates heap memory; performs file I/O
- **Calls:** Not inferable from this file
- **Notes:** Caller owns the returned allocation

### Polyset_CollapseSets
- **Signature:** `polyset_t *Polyset_CollapseSets( polyset_t *psets, int numpolysets )`
- **Purpose:** Merge multiple polysets into a single consolidated polyset.
- **Inputs:** `psets` — array of sets; `numpolysets` — array length
- **Outputs/Return:** Pointer to new single-element `polyset_t` array
- **Side effects:** Likely allocates; may free input
- **Calls:** Not inferable from this file
- **Notes:** Not inferable whether input `psets` is freed

### Polyset_SplitSets
- **Signature:** `polyset_t *Polyset_SplitSets( polyset_t *psets, int numpolysets, int *pNumNewPolysets, int maxTris )`
- **Purpose:** Split polysets so that no individual set exceeds `maxTris` triangles; inverse of collapse.
- **Inputs:** `psets` — source sets; `numpolysets` — count; `pNumNewPolysets` — out-param for new count; `maxTris` — per-set triangle cap (≤ `POLYSET_MAXTRIANGLES`)
- **Outputs/Return:** Newly allocated array of split polysets; `*pNumNewPolysets` written
- **Side effects:** Allocates heap memory
- **Calls:** Not inferable from this file
- **Notes:** Enforces `POLYSET_MAXTRIANGLES` (4096) budget per set

### Polyset_SnapSets
- **Signature:** `void Polyset_SnapSets( polyset_t *psets, int numpolysets )`
- **Purpose:** Snap vertex positions to a grid to eliminate floating-point noise.
- **Inputs:** `psets` — sets to modify in place; `numpolysets` — count
- **Outputs/Return:** void
- **Side effects:** Mutates vertex data in `psets`
- **Calls:** Not inferable from this file
- **Notes:** Snap precision not configurable from the header

### Polyset_ComputeNormals
- **Signature:** `void Polyset_ComputeNormals( polyset_t *psets, int numpolysets )`
- **Purpose:** Calculate and fill per-vertex normals for all triangles in all sets.
- **Inputs:** `psets` — sets to process in place; `numpolysets` — count
- **Outputs/Return:** void
- **Side effects:** Writes into `triangle_t.normals` fields
- **Calls:** Not inferable from this file
- **Notes:** Whether normals are face normals or averaged vertex normals is not inferable from this header alone

## Control Flow Notes
This is a header-only declaration file with no control flow. It is consumed by offline tool pipelines (model exporters, `q3map`, `bspc`) during asset build steps, not during game runtime. The typical usage pattern would be: load → (optionally split/collapse) → snap → compute normals → pass to downstream BSP/model compiler.

## External Dependencies
- `vec3_t` — defined elsewhere (expected in `mathlib.h` or `q_shared.h`)
- `POLYSET_MAXTRIANGLES`, `POLYSET_MAXPOLYSETS` — self-contained constants defined here
- Include guard: `__POLYSET_H__`
- No standard library headers included directly; assumes `vec3_t` is already in scope via the including translation unit
