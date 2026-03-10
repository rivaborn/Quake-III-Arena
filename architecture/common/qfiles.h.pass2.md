# common/qfiles.h — Enhanced Analysis

## Architectural Role

This header is the **binary format contract layer** that decouples offline tool pipelines (q3map, bspc, q3radiant) from the runtime engine (renderer, collision, VMs). Every production asset (BSP map, MD3/MD4 model, QVM bytecode) flows through these exact struct definitions; they are the lingua franca between `code/` and the toolchain, and their stability is mission-critical across 20+ years of modding ecosystems. The `qfiles.h` in `code/` and `common/` must remain byte-for-byte identical by design.

## Key Cross-References

### Incoming (Consumers of Formats Defined Here)
- **Renderer** (`code/renderer/tr_bsp.c`, `tr_model.c`, `tr_init.c`): Reads `dheader_t`, `dsurface_t`, `drawVert_t`, `md3Header_t`, `md4Header_t`; decompresses and uploads to GPU memory
- **Collision System** (`code/qcommon/cm_load.c`): Parses `dheader_t` and all lump structures; builds spatial tree from `dnode_t`, `dleaf_t`; hashes shaders from `dshader_t`
- **QVM Loader** (`code/qcommon/vm.c`, `vm_x86.c`, `vm_interpreted.c`): Validates `vmHeader_t.vmMagic` and version; maps code/data segments
- **Game/cgame VMs** (`code/game/g_spawn.c`, `code/cgame/cg_main.c`): Indirectly consume through engine syscalls; never parse directly
- **q3map, bspc, q3radiant**: Write all formats; use same headers to guarantee binary compatibility

### Outgoing (Dependencies)
- **`q_shared.h`** (included indirectly via translation-unit context): Defines `vec3_t` (float[3]) and `byte` (unsigned char) used throughout
- No function calls—pure compile-time type and constant definitions

## Design Patterns & Rationale

**1. Binary Format as Canonical C Structs**  
Rather than separate text-based format specs, structs *are* the format. This reduces deserialization bugs but requires careful size/alignment control.

**2. Magic Numbers + Versions**  
`VM_MAGIC` (0x12721444), `MD3_IDENT`, `MD4_IDENT`, `BSP_IDENT` + version fields guard against misinterpretation. Versioning allows incremental format evolution (e.g., `BSP_VERSION 46`) without breaking older tools.

**3. Chunked/Lumped Architecture (BSP)**  
The 17-lump design is elegant: `lump_t` pairs (offset, length) allow tools to skip unknown lumps and skip/add future lumps without invalidating old data. This is why Quake III maps remain modifiable 20+ years later.

**4. Coordinate System Anchors**  
World bounds (`MAX_WORLD_COORD ±128*1024`) and lightmap dimensions (128×128) are constants, not dynamic. This ensures all assets quantize to the same grid.

**5. Model Format Evolution**  
MD3 (rigid-body, frame-indexed animation, tags for attaching models) → MD4 (skeletal, bone weights, LODs). Both coexist; the renderer dispatches by format.

**6. Lossy Vertex Compression**  
`md3XyzNormal_t` packs position (int16×3 + scale) + normal (int16 encoded direction) into 8 bytes. `md4Vertex_t` uses full float positions but groups weights variably. Trade-offs explicit in code.

## Data Flow Through This File

**Asset Pipeline (Authoring → Engine)**
```
Level Editor (q3radiant)
  → Map file (MAP format)
    → q3map compiler (reads BSP lump spec from qfiles.h)
      → dheader_t + 17 lumps written to disk (.bsp)
        → Runtime engine loads via cm_load.c + renderer
          → dnode_t/dleaf_t → collision tree
          → dsurface_t → renderer draw calls
          → dshader_t → material binding
```

**Model Pipeline**
```
3D Modeling tool (e.g., Milkshape, Maya)
  → MD3/MD4 export plugin
    → md3Header_t / md4Header_t written to disk (.md3/.md4)
      → Renderer loads via tr_model.c
        → md3Surface_t → meshes per surface
        → md4Frame_t + md4LOD_t → skeletal animation + LODs
```

**QVM Pipeline**
```
C source code (game/cgame/ui)
  → LCC compiler + q3asm (custom Q3 assembler)
    → vmHeader_t + bytecode written (.qvm)
      → qcommon/vm.c validates magic + version
        → VM_Interpret or VM_CallNative
```

## Learning Notes

**Mid-2000s Engineering Decisions:**
- **No struct packing directives.** Modern C would use `#pragma pack(1)` or `__attribute__((packed))`, but Q3 relies on natural alignment and manual padding. Risk: miscompilation on platforms with different alignment rules (e.g., ARM NEON).
- **Fixed limits as compile-time checks.** `MD3_MAX_VERTS 4096`, `MAX_MAP_NODES 0x20000` encode assumptions about memory and performance budgets. Exceed them = silent data corruption. Modern engines use dynamic allocation.
- **BSP coordinate precision.** World bounds ±128K, lightmap per 16 units → 8K×8K worldspace coverage at 128×128 texel lightmaps. Large arena maps risk Z-fighting due to float precision limits in renderer.

**Idiomatic to This Era:**
- **Tag-based attachment system (MD3).** Pre-dating skeletal rigs, tags allowed hierarchical model linking (e.g., attach weapon to player hand). Still used by Q3A player skins.
- **Dual vertex formats.** `drawVert_t` (engine-side interleaved layout) vs. `md3St_t` / `md3XyzNormal_t` (file-side compressed). Post-load, renderer converts to tightly-packed GPU buffers.
- **Immediate enum abuse.** `mapSurfaceType_t` is stored as an int in `dsurface_t`; no validation that the value is a valid enum. Robust code would validate on load.

**Contrast to Modern Engines:**
- Modern engines use glTF 2.0, USD, or proprietary binary formats with built-in versioning, schema validation, and compression.
- No material/shader binding in the models themselves (MD3/MD4 just index). Shaders defined externally in `.shader` files. Decouples content from pipeline.

## Potential Issues

1. **Implicit Struct Alignment.** No `#pragma pack(1)` or `__attribute__((packed))` directives. On strict-alignment platforms (SPARC, old ARM), misaligned reads of e.g. `md3Header_t.ident` could fault or silently read garbage. Cross-platform Q3A ports historically worked around this.

2. **Normal Encoding Lossy.** `md3XyzNormal_t.normal` is a single int16 encoding both direction (lat/long on a sphere) and—implicitly—magnitude ≈1.0. The decoder is not in this file but in `tr_model.c`. If decompression is off, lighting artifacts result.

3. **Coordinate Precision Wall.** BSP planes are `dplane_t { float normal[3], float dist }`. At distance ±128K from origin, float32 precision ≈1 unit. Large architectural features (long corridors) risk Z-fighting or micro-gaps. Modern engines use double-precision or tiling.

4. **No Endianness Markers.** `vmHeader_t` comment says "( dataLength - litLength ) should be byteswapped on load" but there's no magic-number byte-order check. Cross-endian asset loading relies on external tools (not Q3 itself).

5. **No Schema Evolution Guards.** Adding a new lump or extending a struct without a version bump breaks old tools. The comment "must be identical" suggests this was managed by strict CI/version control, not by the format itself.
