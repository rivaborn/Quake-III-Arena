# code/bspc/q3files.h — Enhanced Analysis

## Architectural Role

This file defines the on-disk binary layout for Quake III assets consumed by the **offline BSPC compilation pipeline**. It is a tool-specific variant of the runtime format header (also present in `code/qcommon/qfiles.h`); the Q3_ prefix disambiguates from Quake 1/2/Half-Life BSP formats that BSPC also processes. Data structures here are read by `be_aas_*.c` reachability and clustering functions during offline AAS generation, and written back to disk by the BSPC tool. The file serves as the contract between the BSP compiler and the bot navigation subsystem.

## Key Cross-References

### Incoming (who depends on this file)
- **code/bspc/be_aas_bspc.c**: Drives the entire AAS compilation pipeline; includes q3files.h to parse BSP header and lumps
- **code/bspc/aas_*.c**: Collection modules (area merging, clustering, reachability) consume `q3_dmodel_t`, `q3_dleaf_t`, `q3_dbrush_t`, `q3_dsurface_t` via lumps
- **code/botlib/be_aas_bspq3.c** (runtime): Loads compiled `.aas` files; also deserializes BSP lump data at bot init via `AAS_LoadBSPFile`, `AAS_ParseBSPEntities`
- **q3map/**: Map compiler also consumes these structures to validate BSP output before BSPC processes it

### Outgoing (what this file depends on)
- `vec3_t`, `byte`: Primitives defined in `q_shared.h` (no local includes in q3files.h)
- Implicit dependency on byte-order handling elsewhere: `AAS_SwapAASData` (called by `AAS_LoadAASLump` in `code/botlib/be_aas_file.c` and `code/bspc/aas_file.c`) swaps these struct fields at runtime

## Design Patterns & Rationale

**Lump-Based Format**: The 17-element lumps array (`q3_lump_t`) with offset/length pairs allows:
- Streaming: read header, then seek to specific lumps without parsing everything
- Extensibility: new lumps can be added at the end without breaking older tools
- Partial loading: BSPC can load only geometry-critical lumps (brushes, planes, leaves), skipping shader/lightmap data

**Byte-Offset Indirection** (`ofs*` fields in `md3Surface_t`, `q3_dsurface_t`): Avoids pointers in serialized structures. Offset = (byte position within the lump) + (lump file offset). This is **critical for disk I/O and cross-platform compatibility** — pointers would break when loading on different architectures or into different memory layouts.

**Fixed Capacity Limits** (`Q3_MAX_MAP_*`, `MD3_MAX_*`): Define preallocation sizes; many bspc functions call `AAS_AllocMaxAAS()` with these constants. This was a memory-constrained era design choice — modern engines use dynamic allocation or streaming.

**Version/Ident Pattern** (`Q3_BSP_IDENT`, `Q3_BSP_VERSION`, `MD3_IDENT`, `MD3_VERSION`): Allows toolchain validation. Mismatch detection prevents silent corruption from incompatible tool versions.

**Plane Pair Convention** (comment: "planes (x&~1) and (x&~1)+1 are always opposites"): Encodes front/back plane relationship implicitly in the index. Saves space vs. storing explicit bidirectional links; `AAS_Reachability_*` functions in botlib exploit this invariant.

## Data Flow Through This File

**Offline Compilation Path**:
```
q3map (BSP compiler) 
  → writes code/maps/mapname.bsp 
    (organized as q3_dheader_t + 17 lumps)
        ↓
BSPC tool (be_aas_bspc.c)
  → AAS_LoadBSPFile() 
  → parses q3_dleaf_t, q3_dbrush_t, q3_dsurface_t 
  → AAS_CalcReachAndClusters() 
  → writes mapname.aas (AAS format)
```

**Runtime Bot Loading Path**:
```
Engine startup
  → botlib/be_aas_main.c: AAS_LoadMap()
  → be_aas_bspq3.c: AAS_LoadBSPFile() 
  → parses q3_dheader_t, indexes lumps by Q3_LUMP_* enum
  → caches in global aasworld singleton
  → aasworld.areas, aasworld.areasettings, etc. populated
```

**MD3 Model Loading** (less BSPC-relevant; more runtime):
```
Game entity spawn
  → renderer/tr_model.c or cgame 
  → reads q3_dheader_t + md3Frame_t / md3Surface_t / md3XyzNormal_t
  → populates GPU mesh/skeletal data
```

The **ofs* fields within lumps are critical**: a surface's vertex data is at `(lump[Q3_LUMP_DRAWVERTS].fileofs + dsurface->firstVert)`, not at a direct struct member.

## Learning Notes

**Historical Quake Toolchain Design**: This format reflects the mid-2000s constraint of offline compilation. Modern engines often:
- Stream BSP incrementally or use spatial hashing
- Use container formats (JSON, GLTF, msgpack) for data interchange
- Separate format versioning from tool versioning (semantic versioning)

**Why Q3_ Prefix on Structs**: BSPC also handles Q1/Q2/HL BSP formats — without the prefix collision, including multiple format headers would cause struct name conflicts. The tool swaps between different `MAX_*` limits and different field layouts at compile time.

**Compact Normal Encoding** (`md3XyzNormal_t.normal` as single short): Encodes a 3D unit normal as 2 spherical coordinates packed into 15 bits — saves 6 bytes per vertex. Bot movement and combat rely on decoded normals, so `code/botlib/be_aas_move.c` includes lookup tables.

**Plane Indexing Convention**: The `(x & ~1) + 1` pairing is inlined into reachability functions (`AAS_Reachability_*`) without explicit lookup tables, reflecting instruction-count sensitivity of 2000s compilers.

## Potential Issues

1. **Dead Code**: PCX/TGA struct blocks (lines ~42–72) are commented with broken syntax (`* /` instead of `*/`), making them documentation cruft that blocks immediate readability.

2. **Endianness Implicit**: No `#define` or comment specifying byte order assumptions. Swapping is delegated to `AAS_SwapAASData()` at load time; on big-endian platforms (PowerPC, SGI), this would fail silently if called incorrectly. The first-pass notes this but warrants runtime logging/validation.

3. **Magic Number Density**: `Q3_LUMP_*` enum indices are semantically meaningless (0–16) with no self-documenting names. Tools must maintain a lookup table (implicit knowledge). Modern engines embed lump names in the file header itself.

4. **Array Bounds Not Enforced at Parse Time**: Nothing prevents a malformed BSP from claiming `numVerts = 1000000` when the lump is only 4KB. Overflow corruption occurs in downstream `memcpy` in `AAS_LoadAASLump()` calls; no length validation in `q3files.h` itself (though `be_aas_file.c` does check).
