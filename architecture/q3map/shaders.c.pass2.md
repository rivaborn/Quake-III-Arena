# q3map/shaders.c — Enhanced Analysis

## Architectural Role

This file implements the **shader definition parser and in-memory database for the q3map offline BSP compiler**. Unlike the runtime shader system (`code/renderer/tr_shader.c`), which handles dynamic multi-pass rendering, this module extracts **compile-time material metadata** (surface flags, lighting directives, lightmap sampling hints) that influence lightmapping, visibility processing, and entity placement. The static `shaderInfo[]` lookup table (line 59) is consulted throughout q3map's toolchain to classify surfaces (solid/permeable, light-blocking, shadow-casting) and apply material-driven lighting strategies (backsplash, sun-only forcing, vertex/patch shadow modes).

## Key Cross-References

### Incoming (who depends on this file)
- **q3map core tools** (light.c, vis.c, etc.): Call `ShaderInfoForShader()` to query `surfaceFlags`, `contents`, and `value` during lightmap generation and visibility clustering
- **Global `shaderInfo[]` array** (59): Immutable lookup table populated once at startup; read-only thereafter by all q3map subsystems

### Outgoing (what this file depends on)
- **scriplib** (`LoadScriptFile`, `GetToken`, `MatchToken`): Generic tokenizer for `.shader` script parsing
- **cmdlib** (`FileExists`, `LoadFileBlock`, `DefaultExtension`, `StripExtension`): File I/O and path utilities
- **imagelib** (`LoadTGABuffer`): Read image dimensions and pixel data for color sampling
- **mathlib** (`VectorClear`, `ColorNormalize`, `VectorScale`): Vector math for average-color computation
- **jpeglib** (Windows only, line 220): Fallback JPG loader; Unix builds skip JPG, creating platform-divergence in shader image loading

## Design Patterns & Rationale

**Lazy-Loaded Image Caching**: Images are loaded on first `ShaderInfoForShader()` query (lines 180–214), not at parse time. Missing images degrade to a white 64×64 placeholder (lines 217–223) rather than abort, allowing compilation to complete despite shader art gaps—pragmatic for large projects with incomplete asset pipelines.

**Token-Driven Recursive Descent Parsing**: `ParseShaderFile()` (lines 273–459) uses classic state machines: outer loop tokenizes shader blocks, inner loop accumulates directives until `}`. Nested braces (lines 310–320) skip rendering-pass details (GPU state) in favor of physics/lighting metadata (binary flags). This **deliberate separation** reflects the offline/runtime divide: q3map cares only about walkability, lightmapping strategy, and shadow behavior, not texture blending or vertex deforms.

**Flat Bitmask Architecture**: `infoParms[]` (65–107) maps keywords to bitfield flags, enabling O(n) lookup and bitwise accumulation. The `clearSolid` field (line 330) handles semantic subtleties: e.g., `surfaceparm water` simultaneously sets `CONTENTS_WATER` and clears the default `CONTENTS_SOLID`.

## Data Flow Through This File

1. **Parse phase**: q3map invokes `ParseShaderFile()` once per shader script; directives populate `shaderInfo[]`
2. **Query phase**: Lightmapper, visibility compiler, entity placer call `ShaderInfoForShader()` to classify surfaces by flag membership (e.g., skip `SURF_NODRAW`, detect `q3map_surfacelight` for light entity creation)
3. **Lazy image load**: On first query, `LoadShaderImage()` reads TGA/JPG, samples all pixels to compute average color (used by downstream lightmapping)

**Invariant**: `shaderInfo[]` is immutable post-parse; all subsequent access is read-only.

## Learning Notes

**Historical design vs. modern practice:**
- **No material compilation**: Q3 keeps shaders as plaintext, re-parsed by q3map and the renderer independently. Modern engines serialize materials to binary or use asset pipelines (e.g., SPIR-V, material cache); Q3's approach trades efficiency for simplicity and readability.
- **Offline/runtime duplication**: Both q3map and `code/renderer` parse the same `.shader` files, extracting different subsets (compile-time: physics; runtime: rendering). This separation is intentional—a clean divide between precompiled artifact constraints and runtime rendering flexibility.
- **Flat flag-based classification**: Bitmasks for surface type (walkable, swimmable, etc.) are fast for bitwise queries but inflexible. Modern ECS-based engines use component hierarchies; Q3's design prioritizes compile-time query speed.

## Potential Issues

1. **Buffer overflows** (lines 280, 356, 361): `strcpy()` and `sprintf()` on fixed 1024-byte buffers. No bounds checking; malformed shader paths silently corrupt the stack.
2. **Platform-dependent JPG support** (line 220): JPEG loading is `#ifdef _WIN32` only. Unix builds silently skip JPG images, creating platform-divergent lightmaps if shaders mix TGA and JPG sources.
3. **Memory leaks**: `si->pixels` allocated at line 229/231 but never freed. Acceptable for single-shot q3map tool, but problematic if integrated into long-running editors.
4. **Silent degradation**: Missing shader images (line 210) and invalid surface parms (line 336) emit warnings but continue, risking subtle lightmap and walkability errors that manifest only in-game.
