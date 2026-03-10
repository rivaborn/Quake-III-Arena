# common/trilib.c

## File Purpose
A tool-time (offline/build) library for loading 3D triangle geometry from Alias triangle binary files (.tri format). It parses the proprietary Alias object-separated triangle format and populates polyset arrays used by the Quake III map/model build tools.

## Core Responsibilities
- Parse the Alias binary triangle file format (magic number validation, big-endian byte swapping)
- Handle the hierarchical object/group structure encoded via `FLOAT_START`/`FLOAT_END` sentinel values
- Read per-vertex position, normal, and UV data from disk into `triangle_t` structures
- Allocate and populate `polyset_t` arrays for downstream consumers
- Enforce hard limits on triangle and polyset counts, calling `Error()` on overflow

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vector` | struct | Simple 3-float vector used only in the on-disk representation |
| `aliaspoint_t` | struct | On-disk per-vertex record: normal, position, color, UV |
| `tf_triangle` | struct | On-disk triangle: three `aliaspoint_t` records; direct `fread` target |

## Global / File-Static State
None.

## Key Functions / Methods

### ByteSwapTri
- **Signature:** `static void ByteSwapTri(tf_triangle *tri)`
- **Purpose:** Converts all 4-byte fields in a `tf_triangle` from big-endian (Alias/MIPS disk format) to host byte order.
- **Inputs:** Pointer to an `tf_triangle` read raw from disk.
- **Outputs/Return:** void; modifies `*tri` in place.
- **Side effects:** Overwrites every `int`-sized word in the struct via `BigLong()`.
- **Calls:** `BigLong` (defined in cmdlib)
- **Notes:** Uses a cast to `int*` over the entire struct — relies on the struct being a flat array of 4-byte floats with no padding.

### ReadPolysetGeometry
- **Signature:** `static void ReadPolysetGeometry(triangle_t *tripool, FILE *input, int count, triangle_t *ptri)`
- **Purpose:** Reads `count` raw triangles from `input`, byte-swaps them, and transcribes vertex positions, normals, and UVs into the engine-facing `triangle_t` array starting at `ptri`.
- **Inputs:** `tripool` — base of the triangle pool for bounds checking; `input` — open file; `count` — number of triangles to read; `ptri` — write cursor into the pool.
- **Outputs/Return:** void; fills `ptri[0..count-1]`.
- **Side effects:** Reads from `input`; calls `Error()` on overflow.
- **Calls:** `fread`, `ByteSwapTri`, `Error`
- **Notes:** Color channel (`ptri->colors`) is read from disk but commented out and discarded. Overflow check uses pointer arithmetic against `tripool`.

### TRI_LoadPolysets
- **Signature:** `void TRI_LoadPolysets(const char *filename, polyset_t **ppPSET, int *numpsets)`
- **Purpose:** Top-level entry point. Opens and fully parses an Alias `.tri` file, returning a heap-allocated array of `polyset_t` structures and their count.
- **Inputs:** `filename` — path to the `.tri` file; `ppPSET` — out-param for the polyset array; `numpsets` — out-param for the count.
- **Outputs/Return:** void; sets `*ppPSET` and `*numpsets`.
- **Side effects:** Allocates two heap buffers via `calloc` (polysets + triangle pool); opens and closes a file; calls `Error()` on magic mismatch or capacity overflow; calls `strlwr()` to normalize names in place.
- **Calls:** `fopen`, `fread`, `feof`, `fclose`, `calloc`, `BigLong`, `Error`, `strncpy`, `strcpy`, `strlwr`, `ReadPolysetGeometry`
- **Notes:** The `exitpattern` is the big-endian byte-reversed encoding of `-FLOAT_START`, used to detect end-of-file sentinel. The `texname` field read from the file is parsed but intentionally discarded (commented-out block). The allocated triangle pool (`ptri`) is **never freed** here; ownership transfers implicitly via `pPSET[i].triangles` pointers. Polyset nesting level is tracked by `iLevel` but not otherwise used to enforce logic.

## Control Flow Notes
This file is used exclusively by offline build tools (q3map, model exporters, etc.), not the runtime engine. It has no frame/update/render involvement. `TRI_LoadPolysets` is called once per file during asset import, analogous to a one-shot load function.

## External Dependencies
- `<stdio.h>` — `FILE`, `fread`, `fopen`, `fclose`, `feof`
- `cmdlib.h` — `Error()`, `BigLong()`, `strlwr()`
- `mathlib.h` — `vec3_t` (via polyset.h triangle types)
- `polyset.h` — `triangle_t`, `polyset_t`, `POLYSET_MAXTRIANGLES`, `POLYSET_MAXPOLYSETS`
- `trilib.h` — declares `TRI_LoadPolysets` (defined here)
- `BigLong` — byte-swap utility defined in cmdlib, not in this file
