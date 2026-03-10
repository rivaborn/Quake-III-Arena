# common/l3dslib.c

## File Purpose
Offline tool-time library (not runtime game code) for loading triangle mesh geometry from Autodesk 3DS binary files. It parses the hierarchical chunk-based 3DS format and outputs a flat array of explicit `triangle_t` structs for use by model/map build tools.

## Core Responsibilities
- Open and validate a `.3ds` binary file header
- Recursively parse the 3DS chunk tree, descending into relevant parent chunks (`MAIN3DS`, `EDIT3DS`, `EDIT_OBJECT`, `OBJ_TRIMESH`)
- Parse vertex list chunks (`TRI_VERTEXL`) into a temporary float vertex pool
- Parse face/index list chunks (`TRI_FACEL1`) into a temporary index array
- Convert indexed triangles to explicit (expanded) `triangle_t` structs once both chunks are available
- Return the resulting triangle list and count to the caller via out-parameters

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tri` | struct | Local 4-element index array (v[4]) storing one 3DS face; the 4th index is a flags word in the 3DS format |
| `triangle_t` | typedef (defined in trilib/polyset headers) | Explicit triangle with 3 float vertices, used as the output format |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fverts` | `float[2000][3]` | global (file-scope) | Temporary vertex pool read from `TRI_VERTEXL` chunk |
| `tris` | `tri[MAXTRIANGLES]` | global (file-scope) | Temporary face index list read from `TRI_FACEL1` chunk |
| `bytesread` | `int` | global (file-scope) | Running count of bytes consumed; used to compute chunk-relative offsets |
| `level` | `int` | global (file-scope) | Recursion depth tracker for `ParseChunk` |
| `numtris` | `int` | global (file-scope) | Triangle count in current mesh object |
| `totaltris` | `int` | global (file-scope) | Accumulated output triangle count across all objects |
| `vertsfound` | `int` | global (file-scope) | Flag: vertex chunk has been parsed for current object |
| `trisfound` | `int` | global (file-scope) | Flag: face chunk has been parsed for current object |
| `ptri` | `triangle_t *` | global (file-scope) | Output buffer pointer; allocated once in `Load3DSTriangleList` |

## Key Functions / Methods

### StoreAliasTriangles
- **Signature:** `void StoreAliasTriangles(void)`
- **Purpose:** Expands indexed triangle data into explicit `triangle_t` structs and appends them to the output buffer.
- **Inputs:** Reads from file-scope `tris`, `fverts`, `numtris`, `totaltris`, `ptri`
- **Outputs/Return:** void; writes into `ptri[totaltris..]`; advances `totaltris`; resets `numtris`, `vertsfound`, `trisfound`
- **Side effects:** Modifies all six file-scope counters/flags
- **Calls:** `Error` (from cmdlib)
- **Notes:** Called when both `vertsfound` and `trisfound` are set; the 4th index in `tris[i].v[3]` is silently ignored (3DS face flags)

### ParseVertexL
- **Signature:** `int ParseVertexL(FILE *input)`
- **Purpose:** Reads the `TRI_VERTEXL` chunk body — a count followed by N×3 floats — into the `fverts` pool.
- **Inputs:** `input` — open file at chunk body start; implicit `bytesread`
- **Outputs/Return:** Bytes consumed in this chunk
- **Side effects:** Populates `fverts`; sets `vertsfound`; may call `StoreAliasTriangles`
- **Calls:** `fread`, `feof`, `Error`, `StoreAliasTriangles`
- **Notes:** Errors fatally on duplicate vertex chunks or vertex count > `MAXVERTS` (2000)

### ParseFaceL1
- **Signature:** `int ParseFaceL1(FILE *input)`
- **Purpose:** Reads the `TRI_FACEL1` chunk body — a count followed by N×4 shorts (3 vertex indices + 1 flags word) — into `tris`.
- **Inputs:** `input` — open file at chunk body start
- **Outputs/Return:** Bytes consumed in this chunk
- **Side effects:** Populates `tris`; sets `trisfound`; may call `StoreAliasTriangles`
- **Calls:** `fread`, `feof`, `Error`, `StoreAliasTriangles`
- **Notes:** Errors fatally on duplicate face chunks or face count > `MAXTRIANGLES`

### ParseChunk
- **Signature:** `int ParseChunk(FILE *input)`
- **Purpose:** Reads one 3DS chunk header (type + length), dispatches to a handler or skips, and recursively descends into container chunks.
- **Inputs:** `input` — file positioned at start of a chunk
- **Outputs/Return:** Total bytes consumed (== chunk `length` field)
- **Side effects:** Increments/decrements `level`; advances `bytesread`; reads and discards unknown chunk bodies
- **Calls:** `ParseVertexL`, `ParseFaceL1`, `ParseChunk` (recursive), `fread`, `feof`, `Error`
- **Notes:** Uses `goto ParseSubchunk` / `goto Done` for flow — `EDIT_OBJECT` falls through after reading its null-terminated name string. Container chunks `MAIN3DS`, `EDIT3DS`, `OBJ_TRIMESH` and `EDIT_OBJECT` are parsed recursively; all others are skipped in 4096-byte blocks.

### Load3DSTriangleList
- **Signature:** `void Load3DSTriangleList(char *filename, triangle_t **pptri, int *numtriangles)`
- **Purpose:** Top-level entry point: opens the file, validates its magic, allocates the output buffer, drives the parse, and returns results.
- **Inputs:** `filename` — path to `.3ds` file
- **Outputs/Return:** `*pptri` — heap-allocated `triangle_t` array (caller owns); `*numtriangles` — count
- **Side effects:** Resets all file-scope globals; calls `malloc` for output buffer; calls `fclose`; calls `exit(0)` on open failure or invalid magic
- **Calls:** `fopen`, `fread`, `fseek`, `malloc`, `ParseChunk`, `Error`, `fclose`, `fprintf`, `exit`
- **Notes:** Buffer is always `MAXTRIANGLES * sizeof(triangle_t)` regardless of actual count. An incomplete parse (only vertices or only faces found at end) is treated as fatal via `Error`.

## Control Flow Notes
This file is a **build/offline tool utility** — it is not part of the runtime game loop. It is called by mesh-processing tools (model compilers, map tools) at asset build time. There is no init/frame/shutdown lifecycle; `Load3DSTriangleList` is a one-shot synchronous call.

## External Dependencies
- `<stdio.h>` — `FILE`, `fread`, `fopen`, `fclose`, `fseek`, `feof`, `fprintf`
- `cmdlib.h` — `Error` (fatal error with exit), `MAXTRIANGLES` constant (defined elsewhere in tool lib)
- `mathlib.h` — included but no math functions are directly called here
- `trilib.h` / `l3dslib.h` — declares `triangle_t` type and `Load3DSTriangleList` prototype
- `triangle_t` — defined elsewhere (likely `trilib.h` or a polyset header); **not defined in this file**
- `MAXTRIANGLES` — defined elsewhere in the tool common library; **not defined in this file**
