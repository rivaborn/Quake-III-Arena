# common/bspfile.h

## File Purpose
Declares the shared global BSP map data arrays and counts used by map compilation tools (q3map, bspc, q3radiant), as well as the higher-level `entity_t` / `epair_t` types and the API for loading, writing, and querying parsed BSP/entity data.

## Core Responsibilities
- Extern-declares all flat BSP lump arrays (geometry, visibility, lighting, shaders, etc.) shared across compilation tool translation units
- Provides the `entity_t` and `epair_t` types representing parsed map entities and their key/value metadata
- Declares the BSP file I/O entry points: `LoadBSPFile`, `WriteBSPFile`, `PrintBSPFileSizes`
- Declares the entity-string parse/unparse cycle (`ParseEntities` / `UnparseEntities`)
- Provides key/value query helpers (`ValueForKey`, `FloatForKey`, `GetVectorForKey`, `SetKeyValue`)
- Conditionally includes `qfiles.h` and `surfaceflags.h` from either the tool (`_TTIMOBUILD`) or engine path

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `epair_t` | struct | Singly-linked list node holding one entity key/value string pair |
| `entity_t` | struct | Parsed map entity: world origin, brush list, patch list, draw surface offset, and epair chain |
| `dmodel_t` | struct (from qfiles.h) | Sub-model bounds + surface/brush ranges in the BSP |
| `dshader_t` | struct (from qfiles.h) | Shader name + surface/content flags lump entry |
| `dleaf_t` | struct (from qfiles.h) | BSP leaf: cluster, area, bounds, leaf-surface/brush index ranges |
| `dnode_t` | struct (from qfiles.h) | BSP internal node: plane, children, bounds |
| `dbrush_t` / `dbrushside_t` | struct (from qfiles.h) | Collision brush and its bounding sides |
| `drawVert_t` | struct (from qfiles.h) | Per-vertex position, UVs, lightmap UVs, normal, color |
| `dsurface_t` | struct (from qfiles.h) | Draw surface descriptor: shader, fog, type, vert/index ranges, lightmap metadata |
| `dfog_t` | struct (from qfiles.h) | Fog volume: shader name, brush reference, visible clip side |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `nummodels` / `dmodels` | `int` / `dmodel_t[MAX_MAP_MODELS]` | global (extern) | Sub-model count and array |
| `numShaders` / `dshaders` | `int` / `dshader_t[MAX_MAP_MODELS]` | global (extern) | Shader lump count and array |
| `entdatasize` / `dentdata` | `int` / `char[MAX_MAP_ENTSTRING]` | global (extern) | Raw entity string lump |
| `numleafs` / `dleafs` | `int` / `dleaf_t[MAX_MAP_LEAFS]` | global (extern) | BSP leaf count and array |
| `numplanes` / `dplanes` | `int` / `dplane_t[MAX_MAP_PLANES]` | global (extern) | Plane lump |
| `numnodes` / `dnodes` | `int` / `dnode_t[MAX_MAP_NODES]` | global (extern) | BSP node lump |
| `numleafsurfaces` / `dleafsurfaces` | `int` / `int[MAX_MAP_LEAFFACES]` | global (extern) | Leaf→surface index lump |
| `numleafbrushes` / `dleafbrushes` | `int` / `int[MAX_MAP_LEAFBRUSHES]` | global (extern) | Leaf→brush index lump |
| `numbrushes` / `dbrushes` | `int` / `dbrush_t[MAX_MAP_BRUSHES]` | global (extern) | Brush lump |
| `numbrushsides` / `dbrushsides` | `int` / `dbrushside_t[MAX_MAP_BRUSHSIDES]` | global (extern) | Brush-side lump |
| `numLightBytes` / `lightBytes` | `int` / `byte[MAX_MAP_LIGHTING]` | global (extern) | Lightmap lump |
| `numGridPoints` / `gridData` | `int` / `byte[MAX_MAP_LIGHTGRID]` | global (extern) | Light-grid lump |
| `numVisBytes` / `visBytes` | `int` / `byte[MAX_MAP_VISIBILITY]` | global (extern) | PVS visibility lump |
| `numDrawVerts` / `drawVerts` | `int` / `drawVert_t[MAX_MAP_DRAW_VERTS]` | global (extern) | Draw vertex lump |
| `numDrawIndexes` / `drawIndexes` | `int` / `int[MAX_MAP_DRAW_INDEXES]` | global (extern) | Draw index lump |
| `numDrawSurfaces` / `drawSurfaces` | `int` / `dsurface_t[MAX_MAP_DRAW_SURFS]` | global (extern) | Draw surface lump |
| `numFogs` / `dfogs` | `int` / `dfog_t[MAX_MAP_FOGS]` | global (extern) | Fog volume lump |
| `num_entities` / `entities` | `int` / `entity_t[MAX_MAP_ENTITIES]` | global (extern) | Parsed entity array |

## Key Functions / Methods

### LoadBSPFile
- **Signature:** `void LoadBSPFile( const char *filename )`
- **Purpose:** Reads a `.bsp` file from disk and populates all global lump arrays.
- **Inputs:** Path to the BSP file.
- **Outputs/Return:** void; fills all `d*` globals and their count companions.
- **Side effects:** Allocates/fills global lump arrays; I/O.
- **Calls:** Defined in `common/bspfile.c` (not visible here).
- **Notes:** Must be called before any lump data is accessed.

### WriteBSPFile
- **Signature:** `void WriteBSPFile( const char *filename )`
- **Purpose:** Serializes all global lump arrays into a `.bsp` file on disk.
- **Inputs:** Output file path.
- **Outputs/Return:** void; produces the BSP file.
- **Side effects:** File I/O; reads all global lump arrays.
- **Calls:** Defined in `common/bspfile.c`.
- **Notes:** Caller is responsible for populating lumps before calling.

### ParseEntities / UnparseEntities
- **Signature:** `void ParseEntities(void)` / `void UnparseEntities(void)`
- **Purpose:** Convert between the raw `dentdata` string lump and the structured `entities[]` array.
- **Side effects:** `ParseEntities` populates `entities[]`; `UnparseEntities` serializes back to `dentdata`.

### ValueForKey / SetKeyValue / FloatForKey / GetVectorForKey
- **Purpose:** Query or mutate an entity's epair list by key name.
- **Notes:** `ValueForKey` returns `""` (not NULL) for missing keys, making it safe to use in comparisons without null checks.

### ParseEpair
- **Signature:** `epair_t *ParseEpair(void)`
- **Purpose:** Allocates and returns a single key/value pair parsed from the current script position.
- **Side effects:** Heap allocation; advances global script parser state.

### PrintEntity / PrintBSPFileSizes
- Trivial diagnostic helpers; dump entity epairs or lump byte sizes to the console/log.

## Control Flow Notes
This header is a **build-tool-time** interface (q3map, bspc, q3radiant), not part of the runtime engine. The typical flow is:

1. **Init:** `LoadBSPFile` fills all global arrays from disk.
2. **Process:** Tools read/write the global arrays directly (no frame loop).
3. **Entities:** `ParseEntities` converts the raw entity string to `entity_t[]`; query helpers are used during processing; `UnparseEntities` converts back before writing.
4. **Output:** `WriteBSPFile` flushes all modified lumps back to disk.

## External Dependencies
- `qfiles.h` (via `_TTIMOBUILD` path or `../code/qcommon/qfiles.h`) — defines all `d*_t` BSP lump structs, `MAX_MAP_*` limits, `drawVert_t`, `dsurface_t`, `mapSurfaceType_t`
- `surfaceflags.h` (via `_TTIMOBUILD` path or `../code/game/surfaceflags.h`) — `CONTENTS_*` and `SURF_*` flag bit definitions
- `vec3_t`, `vec_t`, `byte` — defined in `q_shared.h` (pulled in transitively); not defined in this file
- `bspbrush_s`, `parseMesh_s` — forward-declared struct tags; defined in other map-compiler translation units
