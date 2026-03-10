# code/bspc/l_bsp_q1.c — Enhanced Analysis

## Architectural Role

This file implements a Q1 BSP format adapter for the **BSPC offline compiler tool** — a standalone utility (not part of the runtime engine) that converts Quake map files into bot navigation (AAS) data. It bridges the Q1 BSP binary format and the BSPC internal representation by providing load/save serialization, byte-order translation, and entity string marshaling. The file enables BSPC to process Q1 maps for reuse in Quake III, reifying the subsystem's multi-format compilation architecture (Q1, Q2, Q3 via separate loader files like `l_bsp_q1.c`, `l_bsp_q2.c`, `l_bsp_q3.c`).

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — main tool entry point; calls `Q1_AllocMaxBSP()`, `Q1_LoadBSPFile()`, `Q1_FreeMaxBSP()` in load/unload sequence
- **`code/bspc/be_aas_bspc.c`** — AAS compiler; consumes global `q1_d*` arrays after `Q1_LoadBSPFile()` to generate reachability/cluster data

### Outgoing (what this file depends on)
- **`code/bspc/l_bsp_ent.h`** — entity parsing infrastructure; provides `ParseEntity`, `entities[]`, `num_entities` globals shared across all BSP format loaders (Q1/Q2/Q3 converge here)
- **`code/bspc/l_cmd.h`** — low-level file I/O and byte-order utilities: `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `LittleLong`/`LittleFloat`/`LittleShort`, `Error`
- **`code/bspc/l_mem.h`** — memory management: `GetMemory`, `FreeMemory`, `PrintMemorySize`
- **`code/bspc/l_log.h`** — logging: `Log_Print`
- **`code/botlib/l_script.h`** — entity string lexer: `LoadScriptMemory`, `SetScriptFlags`, `FreeScript` (used by `Q1_ParseEntities`)

## Design Patterns & Rationale

### 1. **Max-Capacity Memory Pool Pattern** (`Q1_AllocMaxBSP` / `Q1_FreeMaxBSP`)
The file pre-allocates a single fixed-size heap buffer for each lump type (models, planes, vertices, etc.) at startup. This contrasts with runtime dynamic allocation and reflects **offline tool design philosophy**: all data is loaded into RAM simultaneously for batch processing, and the tool can afford to waste some memory for simplicity. The pattern also ensures predictable failure modes (allocation failure at startup) rather than mid-processing crashes.

**Rationale:** BSPC is a command-line tool without streaming; it processes entire maps. Pre-allocation avoids fragmentation and simplifies error recovery.

### 2. **Byte-Swapping Abstraction** (`Q1_SwapBSPFile`)
A single bidirectional function (`todisk` flag) toggles between disk (little-endian) and host byte orders. This decouples format conversion from load/save logic and enables in-place swapping (no intermediate buffers). The pattern is **ubiquitous in Quake**-era cross-platform code (1990s–2000s), addressing the Intel-vs-PowerPC era.

**Rationale:** Minimal code duplication; obvious intent via `todisk` parameter; proven robustness from original Quake codebases.

### 3. **Lump-Based File I/O** (`Q1_CopyLump` / `Q1_AddLump`)
Each BSP lump (array of structures) is copied as a contiguous block with alignment checks and overflow detection. This mirrors the `.wad` file format and keeps I/O simple. The header is read first, then lumps are pulled from the binary image via offsets, then byte-swapped in-place.

**Rationale:** Quake format design: binary-stable, no parsing overhead, obvious corruption detection (size alignment).

### 4. **Entity Bridge Pattern** (`Q1_ParseEntities` / `Q1_UnparseEntities`)
Entities are stored on disk as a single UTF-8 string (`q1_dentdata`) in key-value pair format (human-readable), but processed as a `entities[]` array in memory. Two functions marshal between formats. This allows the tool to read, edit (via `entities[]`), and write entity data without lossy binary conversion.

**Rationale:** Entity data is semi-dynamic (counts vary per map); string format is portable and debuggable; array format is efficient for in-memory queries.

## Data Flow Through This File

```
┌─────────────────────────────────────────────────────────────┐
│                    LOAD PHASE                               │
├─────────────────────────────────────────────────────────────┤
│ Q1_AllocMaxBSP()                                            │
│   → GetMemory(Q1_MAX_MAP_*) for each lump type             │
│   → Initialize q1_num* counts and q1_allocatedbspmem       │
│                                                              │
│ Q1_LoadBSPFile(filename, offset, length)                   │
│   → LoadFile() → binary image in q1_header                 │
│   → LittleLong swap of header fields                       │
│   → Q1_CopyLump() × 12 → populate q1_d* arrays from image  │
│   → Q1_SwapBSPFile(false) → convert all data to host order │
│   → FreeMemory(q1_header) → discard binary image           │
│                                                              │
│ [Optional] Q1_ParseEntities()                               │
│   → Tokenize q1_dentdata via lexer                         │
│   → Populate entities[] array for in-memory editing        │
└─────────────────────────────────────────────────────────────┘
          ↓ (all global q1_d* arrays ready for AAS compiler)
┌─────────────────────────────────────────────────────────────┐
│              PROCESSING PHASE (in be_aas_bspc.c)            │
├─────────────────────────────────────────────────────────────┤
│ AAS_CalcReachAndClusters() reads q1_d* globals             │
│   → Traverses BSP tree (q1_dnodes, q1_dleafs, q1_dplanes)  │
│   → Generates reach/cluster arrays                         │
└─────────────────────────────────────────────────────────────┘
          ↓ (modified entities[] from parsing)
┌─────────────────────────────────────────────────────────────┐
│                    SAVE PHASE                               │
├─────────────────────────────────────────────────────────────┤
│ [Optional] Q1_UnparseEntities()                             │
│   → Serialize entities[] array → q1_dentdata string        │
│                                                              │
│ Q1_WriteBSPFile(filename)                                  │
│   → Q1_SwapBSPFile(true) → convert data to disk order      │
│   → SafeOpenWrite() → output file                          │
│   → Q1_AddLump() × 12 → append each lump, record offsets   │
│   → fseek() back → overwrite header with final offsets     │
│   → fclose()                                                │
│   [NOTE: data is swapped in-place; globals no longer usable]
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   CLEANUP PHASE                             │
├─────────────────────────────────────────────────────────────┤
│ Q1_FreeMaxBSP()                                             │
│   → FreeMemory(q1_d*) for each lump × 12                   │
│   → Zero all counts and allocatedbspmem                    │
└─────────────────────────────────────────────────────────────┘
```

## Learning Notes

### Idiomatic Quake-Era Patterns

1. **Pre-allocated Max-Capacity Buffers**: Modern engines use growable vectors; Quake (1996–2005) used fixed arrays with `Q3_MAX_MAP_*` constants. This file exemplifies the pattern: budget for worst-case upfront, fail gracefully at load time.

2. **Byte-Order Neutrality**: The `LittleLong` / `LittleFloat` / `LittleShort` macros (defined in `l_cmd.h`) are either no-ops on Intel x86 or active byte-swaps on big-endian (PowerPC). This bidirectional swap pattern is more space-efficient than separate load/save code paths.

3. **Entity String Format**: The `.bsp` entity lump is a single concatenated string of key-value pairs:
   ```
   {
   "classname" "worldspawn"
   "mapname" "q1m1"
   }
   {
   "classname" "light"
   "origin" "0 0 256"
   "light" "200"
   }
   ```
   This is **deliberately human-readable** because mappers and modders edit it with text editors. The same format is reused in Q2/Q3 (see `l_bsp_q2.c`, `l_bsp_q3.c`).

4. **Lump Index Constants**: `Q1_LUMP_MODELS`, `Q1_LUMP_PLANES`, etc. (defined in `l_bsp_q1.h`) provide type-safe access to the header's fixed-size lumps array. This is safer than magic numbers and documents the BSP format.

5. **Multi-Format Tool Architecture**: BSPC demonstrates **format-agnostic compilation**. By providing separate loader files (`l_bsp_q1.c`, `l_bsp_q2.c`, `l_bsp_q3.c`), the tool can process maps from different games without recompiling. The entity parsing layer (`l_bsp_ent.c`) is shared across all formats.

### How This Differs from Modern Practice

- **No dynamic reallocation**: Modern engines use `std::vector<T>` or growable allocators; Quake uses fixed pools.
- **No compression**: Q1 BSP data is uncompressed; modern engines often use ZIP or custom compression (Quake III's `.pk3` is ZIP, but individual lumps are not compressed).
- **No streaming**: All data is loaded into RAM. Modern engines stream large maps.
- **No versioning**: The format is monolithic (`Q1_BSPVERSION`). Modern engines often use versioned asset headers.

### Connection to Engine Subsystems

- **`be_aas_bspc.c` integration**: After `Q1_LoadBSPFile()` populates globals, `AAS_CalcReachAndClusters()` consumes the BSP tree (planes, nodes, leaves) to generate the AAS navigation mesh. This is the core compilation pipeline.
- **Entity bridge**: `Q1_ParseEntities()` outputs to the global `entities[]` array (defined in `l_bsp_ent.h`), which is shared with Q2/Q3 loaders and allows BSPC to manipulate entities uniformly.
- **Byte-order**: The tools run on both x86 (little-endian) and PowerPC (big-endian, especially macOS G5 era), so byte-swap infrastructure is critical.

## Potential Issues

1. **Comma Operator Bug (Line 131)**
   ```c
   q1_allocatedbspmem += Q1_MAX_MAP_EDGES, sizeof(q1_dedge_t);  // WRONG
   // Should be: q1_allocatedbspmem += Q1_MAX_MAP_EDGES * sizeof(q1_dedge_t);
   ```
   The comma operator evaluates `Q1_MAX_MAP_EDGES` (discards result), then evaluates `sizeof(q1_dedge_t)` (returns ~8). So only 8 bytes are counted in the allocation log, not the full edge buffer size. **Impact**: Misleading memory usage reporting; no functional crash (memory is still allocated correctly by `GetMemory`).

2. **Uninitialized Variable in Error Message (Line 593)**
   ```c
   if (q1_header->version != Q1_BSPVERSION)
       Error ("%s is version %i, not %i", filename, i, Q1_BSPVERSION);  // i is loop variable
   ```
   Should be `q1_header->version` instead of `i`. **Impact**: Confusing error message if version mismatch occurs.

3. **Buffer Overflow Risk in `Q1_UnparseEntities` (Line 605–613)**
   ```c
   // Simplified pseudocode:
   for (i = 0; i < num_entities; i++) {
       strcat(q1_dentdata, "{...}");  // Unsafe strcat with manual ptr tracking
   }
   if (strlen(q1_dentdata) > Q1_MAX_MAP_ENTSTRING)
       Error(...);  // Error AFTER overflow!
   ```
   If entities are large, `strcat` can overflow `q1_dentdata` before the bounds check fires. **Impact**: Heap corruption in pathological maps with huge entity counts.

4. **In-Place Byte-Swap Side Effect**
   `Q1_WriteBSPFile` calls `Q1_SwapBSPFile(true)` and then should not use global `q1_d*` arrays afterward. There's no explicit safeguard (e.g., setting pointers to NULL). **Impact**: If a tool chains multiple save operations, the second will operate on swapped (disk-order) data, producing corrupt output.
