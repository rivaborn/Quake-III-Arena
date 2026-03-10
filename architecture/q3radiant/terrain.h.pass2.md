# q3radiant/terrain.h — Enhanced Analysis

## Architectural Role

This header defines the public API for Quake III's editor-only **terrain mesh system**—a specialized brush type allowing visual sculpting of heightfield geometry within the level editor (`q3radiant`). Terrain meshes are built-time constructs: the editor converts them into regular BSP brushes/faces during map export, so they have **zero runtime presence** in `code/server`, `code/cgame`, or `code/game`. The API bridges the editor's brush selection/manipulation framework with a custom heightfield representation that decouples mesh vertices from rigid brush geometry.

## Key Cross-References

### Incoming (who depends on this file)
- **Editor UI/Command Layer**: Commands like `Terrain_Edit()` are wired to UI buttons/hotkeys; `Terrain_SelectPointByRay()` hooks into viewport mouse interaction
- **Brush/Entity Selection**: Generic editor selection code (in `q3radiant/Brush.cpp`, `SELECT.CPP`) checks `OnlyTerrainSelected()` / `AnyTerrainSelected()` to conditionally dispatch to terrain-specific logic vs. brush logic
- **Texture System**: `q3radiant/TextureLoad.cpp` feeds `qtexture_t*` into `Terrain_SetTexture()`, and texture replacement commands call `Terrain_FindReplaceTexture()`
- **Map I/O**: `q3radiant/MAP.cpp` reads/writes terrain entities via `Terrain_Parse()` and `Terrain_Write()`
- **Viewport Rendering**: `q3radiant/CamWnd.cpp` (3D camera), `XYWnd.cpp` (top-down), `ZWnd.cpp` (side view) call `DrawTerrain()`, `Terrain_DrawCam()`, `Terrain_DrawXY()` each frame
- **Entity/Brush List**: Inspector panels call `UpdateTerrainInspector()` to refresh UI state after edits

### Outgoing (what this file depends on)
- **qtexture_t** from renderer: all terrain faces hold references to texture objects; replacements trigger `Terrain_ReplaceQTexture()`
- **brush_t / brush system**: `AddBrushForTerrain()` creates a brush and links it into the world (`bLinkToWorld` parameter); terrain internally wraps brush-like semantics for serialization
- **Common Editor Types**: `vec3_t`, `texdef_t`, `entity_t`; memory management via editor's hunk/zone allocators
- **Ray Casting**: `Terrain_SelectPointByRay()` uses viewport ray geometry; likely calls into qcommon collision or custom ray tests
- **epair (Entity Key/Value) System**: `Terrain_SetEpair()` / `Terrain_GetKeyValue()` store metadata on terrain entities (e.g., shader name, height scaling)

## Design Patterns & Rationale

1. **Dual Representation**: Terrain meshes exist as both `terrainMesh_t` (editor working copy with vertices, faces, normals) and `brush_t` (for map serialization/import). `Terrain_BrushToMesh()` and `AddBrushForTerrain()` convert between them—allowing seamless integration with the brush-centric editor workflow.

2. **Vertex-Level Editing**: Unlike brushes (plane-based), terrains expose individual vertex (`terrainVert_t`) manipulation via `Terrain_UpdateSelected()`, `Terrain_AddPoint()`, `Terrain_SelectAreaPoints()`. This enables sculpting workflows unavailable in rigid geometry.

3. **Deferred Mesh Rebuild**: `Terrain_Move()` and `Terrain_Scale()` accept optional `bRebuild` flags; skipping rebuild during bulk operations (e.g., multi-vert drags) defers expensive normal/bounds recalculation until the operation completes.

4. **Face-Level Texture Independence**: `terrainFace_t` hold individual texture definitions (`texdef_t`), decoupled from vertex data. `SetTerrainTexdef()` / `RotateTerrainFaceTexture()` manipulate faces in isolation—supporting per-face UV rotation and alignment distinct from the mesh's geometric transform.

5. **Isolated Rendering Pipeline**: Three dedicated draw functions (`DrawTerrain`, `Terrain_DrawCam`, `Terrain_DrawXY`) sidestep the generic brush renderer, enabling custom vertex coloring, wireframe modes, or point clouds—essential for real-time visual feedback during sculpting.

## Data Flow Through This File

**Creation Path:**
- User invokes terrain creation UI → `MakeNewTerrain(width, height, texture)` allocates `terrainMesh_t`, initializes vertex grid
- `AddBrushForTerrain()` wraps it as a brush for world linking and undo/redo integration

**Editing Path:**
- Viewport ray → `Terrain_SelectPointByRay()` → identifies vertex/face under cursor
- User drags → `Terrain_UpdateSelected(vMove)` translates move list, calls `Terrain_CalcNormals()`, rerenders
- Optional transform (scale/rotate) → `Terrain_ApplyMatrix()` or `Terrain_Scale()`

**Persistence Path:**
- Map save → `Terrain_Write()` serializes to `.map` file (geometry, textures, epairs)
- Map load → `Terrain_Parse()` reconstructs from brush entity, calls `Terrain_BrushToMesh()`

**Export Path (Bake-Down):**
- Pre-compile: `Terrain_BrushToMesh()` reverses to brush geometry → standard BSP compiler pipeline
- Runtime: terrain no longer exists; all geometry is static BSP faces

## Learning Notes

- **Editor-Only Paradigm**: This file exemplifies the Q3A design where destructive, non-real-time authoring (terrain sculpting, undo/redo, texture assignment) is cleanly separated from the shipping game. No terrain code leaks into `code/server` or `code/game`.
- **Heightfield vs. Arbitrary Mesh**: The system assumes a **regular vertex grid** (width × height). Unlike modern engines' arbitrary meshes, there's no dynamic topology—vertices remain in grid order, enabling efficient neighbor queries and normal recalculation.
- **Normals & Shading**: `Terrain_CalcNormals()` likely computes per-vertex normals from face topology, enabling smooth-shaded preview in the editor despite the underlying face structure.
- **Texture Coordinates vs. World Space**: Terrain faces maintain both texture UV (`texdef_t`) and world-space geometry. This separation is idiomatic to Q3A's two-phase compile pipeline: editor → `.map` → BSP compiler.
- **Integration with Brush Model**: The `brush_t` wrapper allows terrains to participate in brush selection, undo/redo, and entity linking without requiring the editor's core to know about heightfields. Clean abstraction boundary.

## Potential Issues

- **Memory Lifetime**: No explicit reference counting visible in the header. If multiple systems hold `terrainMesh_t*` pointers, `Terrain_Delete()` could orphan dangling references (though editor undo/redo likely prevents this in practice).
- **Ray-Triangle Intersection**: `RayTriangleIntersect()` is exposed but expensive if called per-frame on dense grids; caching or spatial subdivision not evident.
- **Numeric Precision**: Height-based vertex sculpting can accumulate floating-point error; no tolerance guards visible (though unlikely to matter for offline editing).
