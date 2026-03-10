# common/aselib.h

## File Purpose
Public API header for loading and querying 3D mesh data from ASE (ASCII Scene Export) files, a text-based format used by 3ds Max. It exposes an interface for the build tools (q3map, bspc) to import static and animated mesh geometry for use in map compilation and model processing.

## Core Responsibilities
- Declare the ASE file loader entry point
- Expose surface enumeration (count, name, animation frames)
- Declare the cleanup/free routine for loaded ASE data
- Pull in common tool-layer dependencies (`cmdlib`, `mathlib`, `polyset`)

## Key Types / Data Structures
None defined in this file directly; relies on types from bundled headers.

| Name | Kind | Purpose |
|------|------|---------|
| `polyset_t` | struct (from `polyset.h`) | Named triangle set with material, used as the returned surface data |
| `qboolean` | typedef enum (from `cmdlib.h`) | Boolean flag used in `ASE_Load` parameters |

## Global / File-Static State
None declared in this header. State is presumed to reside in `aselib.c` as file-static globals (not visible here).

## Key Functions / Methods

### ASE_Load
- **Signature:** `void ASE_Load( const char *filename, qboolean verbose, qboolean meshanims )`
- **Purpose:** Parses an ASE file from disk into internal static state.
- **Inputs:** `filename` — path to `.ase` file; `verbose` — enable parse logging; `meshanims` — whether to load per-frame mesh animation data.
- **Outputs/Return:** `void`; results accessible via subsequent `ASE_Get*` calls.
- **Side effects:** Allocates internal storage; populates file-static surface/animation tables in `aselib.c`.
- **Calls:** Not inferable from this file.
- **Notes:** Must be called before any `ASE_Get*` query. Not reentrant — only one file loaded at a time.

### ASE_GetNumSurfaces
- **Signature:** `int ASE_GetNumSurfaces( void )`
- **Purpose:** Returns the count of named surface/mesh objects parsed from the loaded ASE file.
- **Inputs:** None.
- **Outputs/Return:** Integer surface count.
- **Side effects:** None.
- **Calls:** Not inferable from this file.
- **Notes:** Valid only after `ASE_Load`.

### ASE_GetSurfaceAnimation
- **Signature:** `polyset_t *ASE_GetSurfaceAnimation( int ndx, int *numFrames, int skipFrameStart, int skipFrameEnd, int maxFrames )`
- **Purpose:** Retrieves the animation frame array for a surface by index, with frame range filtering.
- **Inputs:** `ndx` — zero-based surface index; `numFrames` — out-param receiving actual frame count returned; `skipFrameStart` / `skipFrameEnd` — frames to trim from start/end; `maxFrames` — upper cap on frames.
- **Outputs/Return:** Pointer to an array of `polyset_t` (one per frame); `numFrames` is set on return.
- **Side effects:** Likely allocates the returned frame array.
- **Calls:** Not inferable from this file.
- **Notes:** Caller is responsible for understanding ownership; freed globally by `ASE_Free`.

### ASE_GetSurfaceName
- **Signature:** `const char *ASE_GetSurfaceName( int ndx )`
- **Purpose:** Returns the name string of a surface by index.
- **Inputs:** `ndx` — zero-based surface index.
- **Outputs/Return:** Pointer to an internal name string; caller must not free or mutate.
- **Side effects:** None.
- **Calls:** Not inferable from this file.

### ASE_Free
- **Signature:** `void ASE_Free( void )`
- **Purpose:** Releases all memory allocated by `ASE_Load` and resets internal state.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Frees file-static allocations in `aselib.c`.
- **Calls:** Not inferable from this file.
- **Notes:** Should be called after all surface queries are complete to avoid leaks.

## Control Flow Notes
This is a build-tool utility, not part of the runtime engine frame loop. Usage pattern: `ASE_Load` → `ASE_GetNumSurfaces` → loop `ASE_GetSurfaceName` / `ASE_GetSurfaceAnimation` → `ASE_Free`. Called during map/model compilation (e.g., from q3map's `misc_model` processing), not during game init or render.

## External Dependencies
- `common/cmdlib.h` — `qboolean`, file I/O utilities, error handling
- `common/mathlib.h` — `vec3_t`, `vec_t` used inside `triangle_t` via `polyset.h`
- `common/polyset.h` — `polyset_t`, `triangle_t` — the primary geometry output type
- Implementation (`aselib.c`) defined elsewhere; all state is opaque to callers
