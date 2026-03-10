# code/cgame/tr_types.h

## File Purpose
Defines the shared renderer interface types used by both the client-game (cgame) module and the renderer. It establishes the data structures and constants that describe renderable entities, scene definitions, and OpenGL hardware configuration.

## Core Responsibilities
- Define render entity types and the `refEntity_t` descriptor passed to the renderer
- Define `refdef_t`, the per-frame scene/camera description
- Define OpenGL capability and configuration types (`glconfig_t`)
- Declare bit-flag constants for render effects (`RF_*`) and render definition flags (`RDF_*`)
- Establish hard limits on dynamic lights and renderable entities
- Define polygon vertex and polygon types for decal/effect geometry

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `polyVert_t` | struct | Single vertex for poly rendering: position, texture coords, RGBA modulation |
| `poly_t` | struct | A shader-mapped polygon: shader handle, vertex count, vertex pointer |
| `refEntityType_t` | enum | Discriminator for render entity kind (model, sprite, beam, lightning, portal, etc.) |
| `refEntity_t` | struct | Complete per-entity render submission: type, transform, frame, skin, shader overrides, sprite params |
| `refdef_t` | struct | Per-frame scene descriptor: viewport, FOV, view transform, time, area mask, deform text |
| `stereoFrame_t` | enum | Stereo rendering eye selection (center/left/right) |
| `textureCompression_t` | enum | Texture compression capability (none or S3TC) |
| `glDriverType_t` | enum | GL driver integration level (ICD, standalone, Voodoo) |
| `glHardwareType_t` | enum | GPU capability class with known per-hardware workaround flags |
| `glconfig_t` | struct | Immutable post-init snapshot of all OpenGL capabilities, resolution, and feature flags |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining types and preprocessor constants.

## Control Flow Notes
This header sits at the boundary between the cgame VM and the renderer. `refEntity_t` instances are built each frame by cgame and submitted via the renderer syscall (`RE_AddRefEntityToScene`). `refdef_t` is submitted once per frame via `RE_RenderScene`. `glconfig_t` is populated once during renderer initialization (`R_Init`) and queried read-only thereafter by cgame via `trap_R_GetGlconfig`.

## External Dependencies
- **Defined elsewhere:** `vec3_t`, `qhandle_t`, `qboolean`, `byte`, `MAX_STRING_CHARS`, `BIG_INFO_STRING`, `MAX_MAP_AREA_BYTES` — all from `q_shared.h`
- Driver name macros (`_3DFX_DRIVER_NAME`, `OPENGL_DRIVER_NAME`) conditionalized on `Q3_VM` and `_WIN32` platform defines
- `MAX_DLIGHTS 32` is a hard architectural limit because dlight influence is stored as a 32-bit surface bitmask; `MAX_ENTITIES 1023` is constrained by drawsurf sort-key bit packing in the renderer
