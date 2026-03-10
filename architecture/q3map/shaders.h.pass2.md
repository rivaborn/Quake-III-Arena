# q3map/shaders.h — Enhanced Analysis

## Architectural Role

This header defines the compile-time shader metadata structure used exclusively by the **q3map offline BSP compiler** during the lighting and finalization phase. It bridges shader definition parsing, per-surface lighting computation parameters, and baked-in image data (lightmaps, editor thumbnails). Unlike the runtime `tr_shader.c` system in the renderer, this structure is ephemeral—constructed only during map compilation and never shipped in the final BSP.

## Key Cross-References

### Incoming (who depends on this file)

- **q3map/shaders.c** — Implements `LoadShaderInfo()` and `ShaderInfoForShader()` to parse shader definitions and populate the struct
- **q3map/light.c, q3map/lightmaps.c, q3map/lightv.c** — Query lighting parameters (`lightmapSampleSize`, `lightSubdivide`, shader flags like `forceTraceLight`, `patchShadows`, `forceSunLight`) to control per-surface light baking
- **q3map/light_trace.c** — Uses `lightFilter` flag to determine whether light rays should test against a filter image
- **q3map/surface.c** — Accesses texture/image fields (`lightimage`, `editorimage`, pixels, color, averageColor`) to extract surface properties
- **q3map/mesh.c, q3map/patch.c** — Query `subdivisions` and lighting flags for surface tessellation and shadow casting

### Outgoing (what this file depends on)

- **q3map/shaders.c** — Reads `.shader` text files from the virtual filesystem (via `FS_ReadFile` abstracted at a higher level)
- **q3map/light.c** — Populates struct fields (color, averageColor) by sampling texture pixels
- Implicitly depends on **q_shared.h** types (`vec3_t`, `qboolean`, `MAX_QPATH`) and **code/qcommon/qfiles.h** conventions

## Design Patterns & Rationale

**Monolithic Configuration Struct**: A single struct encapsulates three orthogonal concerns:
  - Shader identity & type flags (`shader`, `backShader`, `flareShader`, `hasPasses`, `globalTexture`, `twoSided`, `autosprite`)
  - **Lighting & tessellation directives** (`lightSubdivide`, `lightmapSampleSize`, `subdivisions`, `forceTraceLight`, `forceVLight`, `vertexShadows`, `patchShadows`, `forceSunLight`, `vertexScale`)
  - **Baked image data** (`lightimage`, `editorimage`, pixels, width, height, color, averageColor)

This tight coupling is typical of pre-ECS game pipelines where offline tools mixed parsing, computation, and output into a single monolith. The lighting flag density (8 distinct shadow/light mode booleans) reflects the complexity of Quake III's multi-algorithm lighting pipeline (trace-based, vertex light, light grids).

## Data Flow Through This File

1. **Input**: `LoadShaderInfo()` parses `.shader` text files, populating struct fields from key-value pairs (e.g., `tesssize`, `q3map_lightSubdivide`, `q3map_lightimage`)
2. **Transformation**: Lighting subsystems query flags and parameters; image loading code reads texture files and computes color statistics
3. **Output**: Struct instances are passed to baking algorithms (light trace, lightmap generation, surface finalization)
4. **Lifetime**: Ephemeral—exists only during compilation; final map BSP contains only static per-surface lightmap data and compiled geometry

## Learning Notes

**Idiomatic to this era**:
- **Offline tool as runtime**: The struct lives in compiler code, not the engine proper. The runtime renderer has an entirely separate shader parsing system (`tr_shader.c` uses a different `shaderStage_t` pipeline).
- **Lighting-centric design**: Quake III's lighting system is multimodal (trace-based radiosity, vertex light grids, light entities, lightmaps). This struct captures the *per-surface override policy* that lets mappers control which algorithm dominates each surface.
- **No separate lighting config**: Unlike modern engines (which externalize lighting parameters in `.exr`, `.hdr`, or layered data files), Quake III bakes lighting directives directly into shader definitions.

**Modern engines differ**:
- Separate **asset metadata** (JSON/YAML) from **scene configuration**
- **Offline baking** outputs to standardized formats (OpenEXR, etc.) rather than embedding in geometry
- **Shader definition** (GPU program) is decoupled from **material parameters** (textures, scalars), which are decoupled from **lighting directives** (bake settings)

## Potential Issues

- **Implicit coupling**: The struct simultaneously serves as a parse target, a configuration container, and a computation workspace. Changes to one concern (e.g., adding a new lighting mode) require editing the same struct.
- **Image data bloat**: Storing `pixels`, `width`, `height`, and color statistics in-memory during compilation can consume significant RAM on large maps with many unique shaders.
- **Unused at runtime**: All this metadata is discarded after compilation; the runtime engine re-parses shaders from text files and builds its own `shaderStage_t` chains, duplicating effort.
