# q3asm/qfiles.h — Enhanced Analysis

## Architectural Role

This file is the **canonical binary protocol header** bridging the offline toolchain (q3asm, q3map, bspc, q3radiant) to the runtime engine. It defines the serialized on-disk format for four major asset types: QVM bytecode, skeletal meshes (MD3/MD4), BSP worlds, and raster images (PCX/TGA). Each format is versioned and magic-number-protected to prevent silent corruption across tool/engine mismatches.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/qcommon/vm.c`**: loads `vmHeader_t`, parses code/data lumps, dispatches to JIT or interpreter
- **`code/renderer/tr_model.c`**: loads `md3Header_t`, interpolates skeletal frames, renders surfaces
- **`code/cgame/cg_players.c`**: consumes md3/md4 player models, attaches weapons/flags via tag system
- **`code/qcommon/cm_load.c`**: parses `dheader_t` and all BSP lumps (dleaf_t, dnode_t, dsurface_t, etc.) to build collision world
- **`code/renderer/tr_bsp.c`**: traverses BSP tree, culls leaves via PVS, sorts surfaces by shader/fog
- **`q3map/`, `code/bspc/`**: write these formats during offline compilation; tool-side mirrors exist in `common/` and `code/bspc/qfiles.h`
- **`code/renderer/tr_image.c`**: loads PCX/TGA for texture rasterization

### Outgoing (what this file depends on)
- None: This is a pure data definition header with no function calls or external dependencies. It defines structure layouts only.

## Design Patterns & Rationale

**Binary Protocol with Versioning**
- Each major format (`vmHeader_t`, `md3Header_t`, `md4Header_t`, `dheader_t`) includes an `ident` magic number (e.g., `0x12721444` for VM) and `version` field
- Prevents silent data corruption from format mismatches or tool/engine version skew
- Magic numbers use unusual bit-shifting (e.g., `('P'<<24)+('S'<<16)+('B'<<8)+'I'` → "IBSP" little-endian) to encode ASCII strings in integers—compact and checksummable

**Offset-based Layout Over Pointers**
- Structures use `fileofs` (byte offset from lump start) and `filelen` (byte count) instead of pointers
- Enables safe binary serialization: pointers would be invalid when loaded from disk
- Each lump is independently loadable; BSP lumps are indexed by enum (`LUMP_ENTITIES`, `LUMP_SHADERS`, etc.)
- Allows streaming and partial loading without address-space assumptions

**Multi-LOD (Level-of-Detail) Architecture**
- MD4 format explicitly supports `md4LOD_t[]` hierarchy: surfaces and bones can differ per LOD, reducing draw cost
- MD3 uses implicit single LOD (though format can support multiple)
- BSP has no LOD concept (world is monolithic)

**Cross-Platform Data Integrity**
- Comment notes byteswapping requirement: `litLength` field indicates which data regions need byte-order correction on load
- Structures are tightly packed and don't rely on compiler padding
- `vec3_t` and `vec2_t` are assumed to be three/two consecutive `float`s

## Data Flow Through This File

1. **Tool → Disk**: q3map/bspc/q3asm write binary files in these formats (structures used as templates for `fwrite`)
2. **Disk → Engine**: `FS_ReadFile` loads entire file into hunk; engine code casts/indexes lumps using offsets
3. **Parsing**: Code like `cm_load.c` iterates lump arrays (e.g., `(dleaf_t *)((byte *)header + lumps[LUMP_LEAFS].fileofs)`)
4. **Transformation**: Loaded structures are often copied into optimized runtime representations (e.g., renderer prebakes BSP visibility, caches vertex data in `tess` buffer)
5. **Lifecycle**: Most data persists for entire map/game session; VM headers are parsed once at load, mesh headers cached in `images_t` hash table

## Learning Notes

**Game Engine Protocol Design (2005 Era)**
- This predates JSON, Protocol Buffers, and modern serialization. The binary approach minimizes parsing overhead—critical for per-frame mesh loading and map parsing on 2005-era hardware.
- Contrast with modern engines (Unreal, Unity) which use text-based or managed serialization.

**Idiomatic Patterns**
- **Magic number as checksum**: Fast validation without CRC/hash overhead
- **Version field as escape hatch**: Allows format evolution without breaking compatibility (e.g., v1 vs v46 BSP)
- **Lumps as flexible array members**: No fixed array sizes; lumps can grow independently

**Cross-Cutting Complexity**
- The "identical in quake and utils directories" comment (line 27) reveals a critical **source-of-truth problem**: this file must be kept in sync across three trees (code/, common/, q3asm/) to prevent tool/engine incompatibility
- Modern solutions (shared submodule, generated headers) didn't exist in the Q3A codebase

**Model Pipeline Insight**
- MD3 → MD4 evolution is visible here: MD4 adds bone weighting and LODs, reflecting real-time skeletal animation becoming standard by TA expansion
- The `md3Tag_t` system (attach points for weapons/flags) is a lightweight attachment mechanism predating modern skeletal sockets

## Potential Issues

**Format Brittleness**
- Adding new lumps to BSP requires incrementing `HEADER_LUMPS` (17), recompiling tools and engine—coupling is tight
- `MAX_QPATH` (64) is hardcoded; exceeding it silently truncates shader/model names (no bounds checking in load code)

**Endianness Assumptions**
- Magic numbers encode ASCII as little-endian (`('P'<<24)...`); engines on big-endian systems must byteswap on comparison
- The `litLength` field is a fragile heuristic for selective byteswapping; modern code would use explicit flags

**No Checksum or CRC**
- Corrupted binary assets are detected only by magic/version mismatches or crash-on-parse, not content integrity
- No recovery mechanism if disk corruption occurs mid-lump

**Pointer Aliasing in Parsing**
- Casting lump data as `(dleaf_t *)base` assumes tight packing and correct alignment; misalignment would silently read garbage
- Modern code would use explicit memcpy to avoid undefined behavior
