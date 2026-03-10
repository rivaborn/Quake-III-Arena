# code/bspc/l_bsp_q2.h — Enhanced Analysis

## Architectural Role

This header defines the **Q2 BSP format bridge layer** within the BSPC offline compilation pipeline. It serves as the public interface through which the main BSPC tool and the embedded botlib/AAS subsystem load and manipulate Quake II BSP map data. The declared globals form a shared **intermediate representation** (after file parsing but before AAS geometry extraction), analogous to an AST in a compiler—all downstream AAS compilation code (`aas_*.c`, `be_aas_*.c`) reads from these lump arrays to extract navigation geometry, entity landmarks, and reachability data.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/bspc/bspc.c`** (main tool entry): Calls `Q2_AllocMaxBSP`, `Q2_LoadBSPFile`, `Q2_PrintBSPFileSizes` to initialize the compilation pipeline
- **`code/bspc/aas_*.c`** (AAS geometry pipeline): Read all `d*` lumps (planes, vertices, faces, brushes, leafs, areas, areaportals) during BSP-to-AAS extraction
- **`code/bspc/be_aas_bspc.c`** (botlib integration stub): Reuses `AAS_LoadBSPFile` and entity parsing to initialize the AAS world
- **`code/bspc/_files.c`** or other I/O layers: May call `Q2_WriteBSPFile` for output BSP writing
- **Entity compilation subsystem** (implicit in `Q2_ParseEntities`/`Q2_UnparseEntities`): Converts raw entity string lump into game-engine-consumable entity state

### Outgoing (what this file depends on)

- **BSP type definitions**: Expects `dmodel_t`, `dleaf_t`, `dplane_t`, `dnode_t`, `dface_t`, `dedge_t`, `dbrush_t`, `dbrushside_t`, `darea_t`, `dareaportal_t`, `dvis_t`, `texinfo_t` to be defined elsewhere (likely a shared Q2 format header, possibly `aasfile.h` or imported from `q3files.h`)
- **`MAX_MAP_*` constants**: Defines allocation ceilings (e.g., `MAX_MAP_FACES`, `MAX_MAP_BRUSHES`)
- **Visibility compression library** (implicit in `Q2_DecompressVis`/`Q2_CompressVis`): Uses RLE-style PVS bitfield codec specific to Q2 format
- **Memory allocator** (implicit in `Q2_AllocMaxBSP`): Likely calls `malloc`-style routines or engine's hunk allocator

## Design Patterns & Rationale

### Global Lump Array Pattern
All BSP data is stored in **file-scoped global pointers**, each paired with a count (`numX`/`dX`). This mirrors the **load-then-process pattern** typical of offline tools:
1. Allocate max-sized buffers once (`Q2_AllocMaxBSP`)
2. Load entire BSP into globals (`Q2_LoadBSPFile`)
3. Multiple subsystems read from globals (AAS geometry extraction, entity parsing, etc.)
4. Free all at shutdown (`Q2_FreeMaxBSP`)

**Rationale**: Avoids parameter threading; simpler than a context struct for a single-map-at-a-time compiler. Contrast with runtime engine's `CM_*` (collision model), which also uses globals but must support multiple maps.

### Multi-Format Support
The `Q2_*` prefix suggests **parallel headers** (`l_bsp_q1.h`, `l_bsp_q3.h`, etc.) each declaring format-specific loaders. BSPC likely conditionally includes the appropriate header based on a build-time or runtime format flag. This is an early form of **polymorphic loading** without virtual functions.

### Visibility Compression Abstraction
`Q2_DecompressVis` / `Q2_CompressVis` encapsulate the Q2-specific RLE compression scheme, isolating format-specific codec logic from callers. The `dpop[256]` lookup table suggests a byte-level decompression acceleration structure.

## Data Flow Through This File

```
[Q2 BSP File on Disk]
         ↓
    Q2_LoadBSPFile(filename, offset, length)
    ├─ Parse file header, seek to lumps
    ├─ Allocate & read each lump into d* globals:
    │  ├─ dmodels[], dplanes[], dvertexes[], ...
    │  ├─ dvisdata[] + visibility header → dvis
    │  ├─ dlightdata[], dentdata[]
    │  └─ Visibility decompression via Q2_DecompressVis
    ↓
[All d* globals populated with map data]
         ↓
[Downstream: AAS Compilation Pipeline]
├─ aas_create.c reads d* lumps, extracts navigable space
├─ aas_reach.c traces jumps/ladders using dfaces, dbrushes, dplanes
├─ be_aas_cluster.c partitions areas into PVS clusters using dvisdata
└─ be_aas_route.c caches routing using darea/dareaportal topology
         ↓
[Q2_WriteB SPFile] — writes modified BSP back (less common)
         ↓
[Output BSP or AAS file]
```

**Entity handling special case**: `dentdata` is a raw string lump (e.g., `"{\n"classname" "info_player_deathmatch"\n...}\n..."`). `Q2_ParseEntities` converts to internal entity array; `Q2_UnparseEntities` does the reverse. Used by AAS for spawn-point and goal recognition.

## Learning Notes

### What This Teaches
1. **Offline tool architecture**: Global state, batch allocation, sequential processing — much simpler than runtime engines needing live hot-swaps
2. **Format abstraction**: Q2/Q3/Q1 loaders are sibling headers with identical logical structure but different binary formats
3. **Visibility in Q2**: Uses RLE-compressed PVS clusters; contrast with Q3's simpler raw-bit PVS in runtime collision model (`qcommon/cm_load.c`)
4. **Entity strings as data**: Maps store all non-geometry data (spawn points, item locations, teleporters) as text key–value pairs; parsing this is critical for AAS goal generation

### Era-Specific Patterns
- **No dynamic allocation per-entity**: All lumps pre-sized to `MAX_MAP_*` limits; no resizable vectors or linked lists within the lump data itself
- **Global state as acceptable**: 2000s era; modern engines favor context structs or dependency injection
- **Format-aware tools**: Separate BSPC versions (or conditional logic) for each game's BSP format; modern level editors (Unreal, Unity) use engine-native formats

### Connections to Engine Subsystems
- **Runtime analog**: `qcommon/cm_load.c` loads Q3 BSP into runtime collision model; uses similar lump-array globals but with different purposes (PVS culling vs. navigation)
- **AAS is unique to Q3 tech**: Quake II's bot AI was simpler; Q3 added the full AAS pathfinding layer (no runtime equivalent in Q2)
- **Entity parsing in game VM**: `code/game/g_spawn.c` parses `dentdata` at runtime; BSPC does it offline for AAS generation

## Potential Issues

- **No format validation**: Callers must trust `Q2_LoadBSPFile` to reject corrupted/wrong-format files gracefully; unclear if bounds-checks exist on lump indices
- **Single-map-at-a-time limitation**: Global state prevents processing multiple Q2 maps in parallel or sequentially within one BSPC run (would require a context struct or hash table keyed by filename)
- **Visibility codec assumption**: If Q2 uses a different RLE variant than expected, decompression will silently produce garbage; no magic-number validation visible here

---
