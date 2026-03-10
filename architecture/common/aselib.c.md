# common/aselib.c

## File Purpose
Parses Autodesk ASCII Scene Export (ASE) 3D model files used by Quake III Arena's toolchain. It loads mesh geometry, materials, and animation frame sequences from ASE text format into in-memory structures, then exposes them as `polyset_t` arrays for consumption by downstream tools (model compilers, map tools).

## Core Responsibilities
- Read and tokenize an ASE file from disk into a flat memory buffer
- Parse the ASE hierarchy: `MATERIAL_LIST` → `GEOMOBJECT` → `MESH` / `MESH_ANIMATION`
- Build per-frame mesh arrays (vertices, faces, texture vertices, texture faces)
- Resolve material bitmap paths relative to `gamedir`, normalizing path separators
- Filter/discard objects named "Bip", "ignore_", or improperly labeled bodies when grabbing animations
- Convert parsed mesh data into `polyset_t` / `triangle_t` structures for external consumers
- Free all heap-allocated mesh frame data on request

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aseVertex_t` | struct | Single vertex: position (x,y,z), normal (nx,ny,nz), UV (s,t) |
| `aseTVertex_t` | struct | Texture-only vertex: UV (s,t) |
| `aseFace_t` | typedef (`int[3]`) | Triangle index triple for geometry or UV faces |
| `aseMesh_t` | struct | One animation frame's mesh: counts, dynamic vertex/face arrays, parse cursors |
| `aseMeshAnimation_t` | struct | Array of up to `MAX_ASE_ANIMATION_FRAMES` (512) frames + current frame cursor |
| `aseMaterial_t` | struct | Material name string (128 chars) |
| `aseGeomObject_t` | struct | Named surface object: material ref, animation sequence, frame count |
| `ase_t` | struct | Top-level parser state: materials, objects, buffer pointers, flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_token` | `char[1024]` | static (file) | Reusable token scratch buffer filled by `ASE_GetToken` |
| `ase` | `ase_t` | static (file) | Singleton parser state holding all parsed objects/materials and buffer cursor |

## Key Functions / Methods

### ASE_Load
- **Signature:** `void ASE_Load( const char *filename, qboolean verbose, qboolean grabAnims )`
- **Purpose:** Entry point; reads the entire ASE file into a heap buffer, resets global `ase` state, and triggers recursive parsing.
- **Inputs:** File path, verbosity flag, flag indicating whether animation frames should be captured.
- **Outputs/Return:** None (populates global `ase`).
- **Side effects:** Allocates `ase.buffer` via `malloc`; calls `ASE_Process`; prints to stdout.
- **Calls:** `fopen`, `Q_filelength`, `malloc`, `fread`, `fclose`, `ASE_Process`, `Error`.
- **Notes:** Buffer is not freed here; caller must call `ASE_Free`.

### ASE_Free
- **Signature:** `void ASE_Free( void )`
- **Purpose:** Releases all per-frame heap allocations (vertexes, tvertexes, faces, tfaces) for every parsed object.
- **Side effects:** Frees heap memory; zeroes each `aseGeomObject_t` via `memset`.
- **Calls:** `ASE_FreeGeomObject`.

### ASE_GetNumSurfaces
- **Signature:** `int ASE_GetNumSurfaces( void )`
- **Purpose:** Returns the count of successfully parsed geometry objects.
- **Outputs/Return:** `ase.currentObject` (int).

### ASE_GetSurfaceName
- **Signature:** `const char *ASE_GetSurfaceName( int which )`
- **Purpose:** Returns the name string of the nth parsed object, or NULL if it has no frames.

### ASE_GetSurfaceAnimation
- **Signature:** `polyset_t *ASE_GetSurfaceAnimation( int which, int *pNumFrames, int skipFrameStart, int skipFrameEnd, int maxFrames )`
- **Purpose:** Converts an object's animation frames into an array of `polyset_t`, copying vertex positions and UV coordinates into `triangle_t` entries. Supports frame range clamping and skipping.
- **Inputs:** Object index; output frame count ptr; optional skip range (pass -1 to disable); max frame cap (-1 = no cap).
- **Outputs/Return:** `calloc`-allocated `polyset_t[]`; `*pNumFrames` set to actual frame count.
- **Side effects:** Allocates `polyset_t` array and per-frame `triangle_t` arrays via `calloc`. Caller owns memory.
- **Notes:** Does not copy normals. UV `t` component is already flipped (1.0 - v) during mesh parse.

### ASE_Process *(static)*
- **Signature:** `static void ASE_Process( void )`
- **Purpose:** Top-level token dispatch loop; routes `*MATERIAL_LIST` and `*GEOMOBJECT` blocks to sub-parsers; discards skeleton/ignore objects; calls `CollapseObjects` at end.
- **Side effects:** Modifies global `ase`; calls `Error` if no animation data found.
- **Calls:** `ASE_GetToken`, `ASE_SkipRestOfLine`, `ASE_SkipEnclosingBraces`, `ASE_ParseBracedBlock`, `ASE_FreeGeomObject`, `CollapseObjects`.

### ASE_GetToken *(static)*
- **Signature:** `static int ASE_GetToken( qboolean restOfLine )`
- **Purpose:** Advances `ase.curpos` past whitespace, reads the next whitespace-delimited token (or rest of line) into `s_token`.
- **Outputs/Return:** 1 if a token was read, 0 at end of buffer.

### ASE_ParseBracedBlock *(static)*
- **Signature:** `static void ASE_ParseBracedBlock( void (*parser)( const char *token ) )`
- **Purpose:** Consumes a `{ ... }` block, dispatching non-brace tokens to the provided callback. Handles nesting.
- **Notes:** `parser` may be NULL (used to silently skip blocks).

## Control Flow Notes
This file is a **build-time tool library** (used by `q3map`, `bspc`, and model compilers), not part of the runtime game engine. Flow is purely synchronous and linear: `ASE_Load` → `ASE_Process` (recursive descent via `ASE_ParseBracedBlock` callbacks) → consumers call `ASE_GetNumSurfaces`/`ASE_GetSurfaceAnimation` → `ASE_Free`. No frame/update loop involvement.

## External Dependencies
- **`aselib.h`** → pulls in `cmdlib.h` (`Error`, `Q_filelength`, `gamedir`), `mathlib.h` (`qboolean`), `polyset.h` (`polyset_t`, `triangle_t`)
- `gamedir` — global string defined in `cmdlib`; used for material path resolution
- `Error` — fatal error handler defined in `cmdlib`
- `Q_filelength` — file size utility defined in `cmdlib`
- Standard C: `<stdio.h>`, `<stdlib.h>`, `<assert.h>`
