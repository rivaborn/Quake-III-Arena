# common/trilib.h

## File Purpose
Header file declaring the interface for loading triangle/polyset data from Alias triangle files. It exposes a single loading function used by tools (e.g., model compilers) that need to import geometry from the Alias `.tri` format.

## Core Responsibilities
- Declare the public API for the Alias triangle file loader
- Expose `TRI_LoadPolysets` as the sole entry point for consumers of `trilib.c`

## Key Types / Data Structures
None defined in this file. Uses `polyset_t` (defined elsewhere, likely `common/polyset.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### TRI_LoadPolysets
- **Signature:** `void TRI_LoadPolysets( const char *filename, polyset_t **ppPSET, int *numpsets );`
- **Purpose:** Loads one or more polysets (named triangle groups) from an Alias `.tri` file on disk.
- **Inputs:**
  - `filename` — path to the Alias triangle file to load
  - `ppPSET` — out-parameter; receives a pointer to an allocated array of `polyset_t`
  - `numpsets` — out-parameter; receives the count of polysets loaded
- **Outputs/Return:** `void`; results are returned via `ppPSET` and `numpsets`
- **Side effects:** Allocates memory for the `polyset_t` array (caller is responsible for freeing). Performs file I/O.
- **Calls:** Defined in `common/trilib.c`; not visible here.
- **Notes:** The double-pointer pattern (`polyset_t **`) indicates the function allocates the array internally. Passing a null or invalid filename behavior is not inferable from this file.

## Control Flow Notes
Not part of the runtime game loop. Used exclusively at offline tool/build time — model compilers or map-building utilities call this during asset processing to import Alias triangle meshes.

## External Dependencies
- `polyset_t` — struct defined elsewhere (likely `common/polyset.h`)
- Implementation resides in `common/trilib.c`
