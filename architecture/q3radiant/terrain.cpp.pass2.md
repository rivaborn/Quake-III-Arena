# q3radiant/terrain.cpp — Enhanced Analysis

## Architectural Role

This file implements Radiant's terrain mesh editor subsystem, occupying a unique position as a **level-editor-only feature** that bridges two representation paradigms: the core brush-based BSP architecture and an alternative heightmap-grid approach. Terrain meshes are ephemeral level-editor constructs that get serialized into brush geometry for the runtime engine; there is no terrain-specific data in shipped map files or the runtime engine. This file is the sole owner of terrain editing logic and maintains bidirectional coupling with the brush system through "symbiot" pointers (`terrainMesh_t::pSymbiot` ↔ `brush_t::pTerrain`).

## Key Cross-References

### Incoming (who depends on this file)
- **`q3radiant/terrain.h`**: Header exporting public API (`MakeNewTerrain`, `Terrain_Parse`, `AddBrushForTerrain`, `Terrain_Delete`, terrain editing functions)
- **Radiant UI/menu system**: Calls through Radiant's command/menu infrastructure to trigger terrain operations (inferred from `UpdateTerrainInspector()` calls)
- **Brush serialization** (`q3radiant/Brush.cpp`, `MAP.cpp`): Must invoke `Terrain_Parse` during map load to reconstruct terrain meshes; must serialize terrains via brush geometry on save
- **Entity/world system** (`q3radiant/ENTITY.CPP`, `q3radiant/SELECT.CPP`): Terrain meshes linked to world via `Entity_LinkBrush` and `Select_Brush` calls

### Outgoing (what this file depends on)
- **Global brush/world state**: `active_brushes`, `selected_brushes`, `world_entity` (from qe3.cpp)
- **Brush creation/manipulation** (`q3radiant/Brush.cpp`): `Brush_Create`, `Brush_AddToList`, `Brush_Build`, `Entity_LinkBrush`
- **Selection/UI** (`q3radiant/SELECT.CPP`): `Select_Delete`, `Select_Brush`
- **Texture system** (`q3radiant/Textures.cpp`): `Texture_ForName` for texture lookup
- **Parser infrastructure** (`q3radiant/Parsing.cpp`): `GetToken`, `TokenAvailable` for script parsing
- **Warning/logging** (`qe3.cpp`): `Warning` macro for user feedback
- **Global texture state** (`g_qeglobals`): stores `d_terrainWidth`, `d_terrainHeight` editor parameters
- **Memory** (`qe3.cpp`): `qmalloc` for allocation; note use of `reinterpret_cast` to place heightmap after `terrainMesh_t` struct in single allocation

## Design Patterns & Rationale

### Embedded Heightmap Allocation
```cpp
heightmapsize = sizeof(terrainVert_t) * width * height;
size = sizeof(terrainMesh_t) + heightmapsize;
pm = reinterpret_cast<terrainMesh_t*>(qmalloc(size));
pm->heightmap = reinterpret_cast<terrainVert_t*>(pm + 1);
```
Allocates the struct and heightmap in a single contiguous block, reducing allocator pressure and improving cache locality. The pattern `pm + 1` computes the address immediately after the struct header — a C idiom predating modern memory pools.

### Symbiot Coupling Pattern
Terrain meshes and brushes maintain **bidirectional ownership pointers** (`pSymbiot` / `pTerrain`), allowing either representation to query the other. On delete, the symbiot is cleared to prevent dangling pointers. This is acknowledged as a FIXME in the code: `// FIXME: this entire type of linkage needs to be fixed`, suggesting the original authors recognized the fragility of dual-ownership.

### Two Triangles Per Cell
Each heightmap cell generates two triangles with alternating winding (`(x + y) & 1`), producing a checkerboard diagonal pattern. This minimizes degenerate triangles and simplifies the triangle indexing scheme: `index = x + y * pm->width; which = index & 1` maps a global triangle index to a cell and which of the two triangles within that cell.

### Texture-Per-Vertex Semantics
Unlike traditional brushes (texture-per-face), terrain vertices carry texture info and an alpha channel (`rgba[3]`) used for per-vertex texture blending. The `Terrain_GetVert` function computes alpha as 1.0 if the vertex's tri texture matches the query texture, else 0.0 — enabling selective rendering per texture.

## Data Flow Through This File

**Inflow (map load)**:
1. Parser invokes `Terrain_Parse()` during brush deserialization
2. Creates `terrainMesh_t` with width/height/origin from script tokens
3. Reads per-vertex height and texture definition for each heightmap cell
4. Invokes `AddBrushForTerrain()` to materialize a brush wrapper
5. Terrain is linked to world and becomes editable

**In-memory operations**:
- Height/texture/color edits mutate `terrainMesh_t::heightmap` array in-place
- `Terrain_CalcNormals()` / `Terrain_CalcBounds()` precompute derived data for rendering/collision
- Symbiot brush is updated via `Brush_Build()` to reflect changed mesh geometry

**Outflow (map save)**:
- `Terrain_SurfaceString()` serializes terrain surface data to script format
- Radiant's brush/entity save pipeline converts terrain mesh back to brush geometry, writing it as a map entity with custom terrain syntax

**Rendering path** (inferred):
- `Terrain_GetTriangles()` supplies vertices in texture-specific batches for GL drawing in the Radiant viewport

## Learning Notes

### Editor-Engine Boundary
This is a pedagogically important example of a **map representation that exists only in the editor**. The runtime engine has no concept of terrains — they are flattened to brush geometry before shipping. This is unlike modern engines (e.g., Unreal, Unity) where terrain is a first-class runtime asset type. The tradeoff Q3A made: terrain editing in the editor is more specialized (fewer features), but the engine stays simpler.

### Idiomatic 1990s C Patterns
- Manual memory layout with pointer arithmetic (`pm + 1`)
- Struct embedding (`heightmap` directly embedded as pointer post-struct-header)
- Bitmask packing (`(x + y) & 1` for diagonal orientation)
- No explicit constructors/destructors; init functions like `MakeNewTerrain` play that role

### No Validation of Terrain-Brush Consistency
Once a terrain is linked to a brush symbiot, there's no runtime check that they stay synchronized if the user edits the brush directly (e.g., dragging a face). Modifying the symbiot brush without updating the terrain mesh would silently desynchronize them.

## Potential Issues

1. **Symbiot Dangling Pointers**: The code sets `p->pSymbiot->terrainBrush = false` on delete, but if the brush is deleted first, `pm->pSymbiot` becomes dangling. No defensive check prevents re-entrant deletes.

2. **Texture Array Overflow**: `Terrain_AddTexture` checks `pm->numtextures >= MAX_TERRAIN_TEXTURES` and warns but doesn't fail gracefully; the terrain becomes inconsistent if textures exceed the limit.

3. **No Normalization of Scale**: `scale_x` and `scale_y` are parsed as `atoi(token)`, losing fractional precision. If a user manually edits a `.map` file with non-integer scales, they are silently truncated.

4. **Missing Error Handling in Parse**: `Terrain_ParseFace` can fail (returns NULL on missing texture) but callers don't propagate the error; the entire terrain load aborts with `Terrain_Delete` but leaves partial state.

5. **Ephemeral Parsing State**: The parser uses global `token` buffer from the main parser, making recursive or concurrent parse calls unsafe (though Radiant is single-threaded, making this unlikely in practice).
