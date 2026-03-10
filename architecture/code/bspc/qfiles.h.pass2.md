# code/bspc/qfiles.h — Enhanced Analysis

## Architectural Role

This header defines binary file formats for **offline tool processing** in Q3A's BSPC (BSP compiler) pipeline—specifically the Q2-era BSP format used as intermediate input, plus legacy asset formats (MD2, PAK, PCX, SP2, WAL). Unlike the runtime `code/qcommon/qfiles.h` which defines Q3 BSP structures consumed by `CM_LoadMap`, this file bridges **compiled map sources to AAS (Area Awareness System) geometry generation**. BSPC reads Q2/Q3 BSP files using these struct definitions, then feeds the geometry into botlib's reusable AAS compilation pipeline, producing `.aas` files that the runtime botlib loads for bot navigation.

## Key Cross-References

### Incoming (who depends on this file)
- **code/bspc/l_bsp_q2.c** — Reads Q2 BSP files using `dheader_t`, `dmodel_t`, `dface_t`, `dbrush_t`, `dleaf_t`, and all lump structures defined here
- **code/bspc/aas_file.c**, **code/botlib/be_aas_file.c** — Load/validate/serialize AAS geometry computed from BSP input; use lump indices and struct layouts
- **code/botlib/be_aas_bspq3.c** — Parses BSP entities and performs PVS/cluster queries using BSP leaf/node hierarchy (`dleaf_t`, `dnode_t`, `dvis_t`)
- **code/bspc/be_aas_bspc.c** (bspc adapter) — Instantiates botlib's `aasworld` state by loading Q2/Q3 BSP files via these definitions
- **code/bspc/aas_create.c**, **code/bspc/aas_gsubdiv.c**, etc. — Create AAS areas from BSP brushes; use `dbrush_t`, `dbrushside_t`, `dplane_t` to decompose geometry

### Outgoing (what this file depends on)
- **None**. This is a pure header with no `#include` directives. Assumes caller provides `byte`, `short`, `int` type definitions and `MAX_SKINNAME` constant.
- Implicitly ties to **q_shared.h** for `CONTENTS_*` and `SURF_*` flag definitions that must remain synchronized (see comment at line ~360).

## Design Patterns & Rationale

**1. Literal struct-to-disk mapping**  
All `d*_t` structs (prefix `d` for "disk") are packed binary layouts, consumed by `fread`/`fwrite` in tool code. This reflects 1990s Quake-era tooling: no serialization layer, direct memory-mapped I/O. No padding, explicit byte-alignment requirements (e.g., `unsigned short` edges in `dedge_t`, `byte` vertex data in `dtrivertx_t`).

**2. Dual-format legacy architecture**  
This file defines Q2 BSP, but Q3A's runtime uses a different BSP format (in `qcommon/qfiles.h`). BSPC was bootstrapped from Q2 tools and retained Q2 readers for compatibility. The AAS pipeline it feeds is format-agnostic—it accepts Q2 or Q3 BSP input and outputs the same AAS geometry.

**3. Lump-index enumeration pattern**  
`LUMP_*` constants (0–18) and the `lump_t` offset/length pairs in `dheader_t` allow BSP tools to skip/seek directly to any data section without parsing. This modular structure enabled Q2 → Q3 map upgrades by preserving some lumps and recompiling others.

**4. Content/surface flags as a shared constant dictionary**  
`CONTENTS_*` and `SURF_*` flags are defined here (used by BSP compilers and runtime), but the inline comment warns "these definitions also need to be in q_shared.h!" This reflects the engine's modular organization: the same flag vocabulary must be shared across offline tools (`bspc/`), game VM (`code/game/`), and runtime (`code/qcommon/`), but there is no central place—each subsystem has its own copy. Any change requires manual sync across three files.

## Data Flow Through This File

**Input data path:**
- BSP file on disk → `fread` into `dheader_t` → iterate `HEADER_LUMPS` offsets → load each lump (geometry, entities, visibility, etc.)

**Struct hierarchy for geometry:**
```
dheader_t (file root)
├── dmodel_t[MAX_MAP_MODELS]        (submodel AABB + headnode pointer)
├── dplane_t[MAX_MAP_PLANES]        (BSP splitting planes)
├── dnode_t[MAX_MAP_NODES]          (BSP internal nodes, each refs a plane)
├── dleaf_t[MAX_MAP_LEAFS]          (BSP leaf nodes; contain face/brush ranges)
├── dface_t[MAX_MAP_FACES]          (polygons; ref texinfo, edges, lightmap)
├── dedge_t[MAX_MAP_EDGES]          (indexed vertex pairs)
├── dvertex_t[MAX_MAP_VERTS]        (xyz positions)
├── dbrush_t[MAX_MAP_BRUSHES]       (convex volumes)
├── dbrushside_t[MAX_MAP_BRUSHSIDES] (brush bevels; ref plane)
├── texinfo_t[MAX_MAP_TEXINFO]      (texture projection, animation chain)
├── dvis_t header + bitoffsets      (PVS/PHS visibility clustering)
└── raw lighting, entities (ASCII strings)
```

**AAS compilation pipeline:**
1. BSPC loads BSP using `dheader_t` + lump iteration → `aasworld` singleton
2. botlib's `AAS_CreateMapBrushes` decomposes `dbrush_t` → AAS areas
3. `AAS_InitReachability` tests inter-area movement (jump arcs, ladder climbing) using brush geometry + `dplane_t` collision
4. Final AAS file written to disk by `AAS_WriteAASFile` (uses a different binary format, not defined in this header)

## Learning Notes

**Legacy format archaeology:**  
Developers studying Q3A should understand this file represents the **Q2 era** (1997), not Q3 (1999). Q3A's actual map format uses different BSP structures in `qcommon/qfiles.h`. This dual-format design reflects id Software's modular tool architecture: BSPC is a standalone compiler that can ingest either Q2 or Q3 BSP and produce AAS geometry.

**Lump-based modularity:**  
The `lump_t` and `LUMP_*` pattern (fixed-size header with offset/length pairs for each section) became ubiquitous in game engines—it allows incremental recompilation. For example, a light-only rebuild only rewrites the `LUMP_LIGHTING` section without touching geometry.

**Flag replication problem:**  
The comment at line ~360 (`// these definitions also need to be in q_shared.h!`) illustrates a classic versioning problem: when content/surface flags change, three files must be manually kept in sync, with no compiler enforcement. Modern engines use a single authority source or code generation to avoid this.

**Packed binary structs in C:**  
All `d*_t` typedefs rely on C struct layout guarantees. On a 32-bit x86 platform (the target era, ~2000), this is straightforward, but porting to 64-bit or other architectures requires `#pragma pack(1)` or careful member ordering. The code likely doesn't do this—it's another artifact of its age.

## Potential Issues

**1. Flag synchronization burden:**  
`CONTENTS_*` and `SURF_*` must be manually kept in sync with `code/qcommon/q_shared.h` and `code/game/q_shared.h`. There is no build-time check or `#include` guarantee. A developer adding a new flag to one copy may silently forget the others, causing runtime collisions or behavior divergence between tools and engine.

**2. Platform-dependent struct layout:**  
No explicit `#pragma pack` directives. On platforms with different struct alignment (e.g., ARM, MIPS, or 64-bit), binary BSP files may not deserialize correctly. The code likely assumed little-endian x86 exclusively.

**3. MAX_* bounds not enforced:**  
Constants like `MAX_MAP_VERTS` (65536) are advisory; a malformed BSP file with `num_verts > 65536` will not trigger a compile-time error. Runtime buffer overflows are possible if BSPC doesn't validate lump sizes against these limits. (The cross-reference shows `be_aas_bspc.c` does have some validation, but it's not universal.)
