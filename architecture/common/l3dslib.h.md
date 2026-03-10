# common/l3dslib.h

## File Purpose
Public header for the 3DS (3D Studio) file loader library. Exposes a single function for importing triangle geometry from Autodesk 3DS format files, used by map/model build tools.

## Core Responsibilities
- Declare the public API for loading triangle mesh data from `.3ds` files
- Bridge between the 3DS binary format and the engine's internal `triangle_t` representation

## Key Types / Data Structures
None defined in this file.

| Name | Kind | Purpose |
|------|------|---------|
| `triangle_t` | struct (typedef, defined elsewhere) | Engine triangle type; pointer-to-pointer used as output buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### Load3DSTriangleList
- **Signature:** `void Load3DSTriangleList(char *filename, triangle_t **pptri, int *numtriangles);`
- **Purpose:** Loads all triangle geometry from a 3DS file into a dynamically allocated array of `triangle_t`.
- **Inputs:**
  - `filename` — path to the `.3ds` file to load
  - `pptri` — out-pointer; receives the address of the allocated triangle array
  - `numtriangles` — out-pointer; receives the count of triangles loaded
- **Outputs/Return:** `void`; results delivered via `pptri` and `numtriangles` out-parameters
- **Side effects (global state, I/O, alloc):** File I/O; heap allocation for the triangle array (caller is responsible for freeing, inferred from pattern)
- **Calls:** Not inferable from this file (implementation in `common/l3dslib.c`)
- **Notes:** `pptri` is a double-pointer, so the callee allocates the buffer and writes the pointer back to the caller. Caller must not pass `NULL` for either output parameter.

## Control Flow Notes
Used exclusively at asset-build / tool time (q3map, q3radiant model import). Not part of the runtime game loop. Called once per 3DS asset file during model or level compilation.

## External Dependencies
- `triangle_t` — defined elsewhere (likely `common/trilib.h` or `common/polyset.h`)
- No standard library headers included directly in this header
