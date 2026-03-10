# code/bspc/l_bsp_hl.c — Enhanced Analysis

## Architectural Role
This file implements the Half-Life (GoldSrc) BSP format adapter for BSPC, one of five parallel format handlers (`l_bsp_q1.c`, `l_bsp_q2.c`, `l_bsp_q3.c`, `l_bsp_hl.c`, `l_bsp_sin.c`) in the offline tool suite. It translates GoldSrc BSP geometry into the shared `entities[]` array via `HL_ParseEntities`/`HL_UnparseEntities`, enabling BSPC to accept Half-Life maps as input for conversion to other formats or AAS compilation. The module operates entirely offline—allocate-max on startup, load one map, convert/process, write output, then free all.

## Key Cross-References

### Incoming (BSPC conversion pipeline)
- **bspc.c** or equivalent BSPC dispatcher likely calls:
  - `HL_AllocMaxBSP()` during initialization
  - `HL_LoadBSPFile(filename, offset, length)` to ingest a Half-Life BSP
  - `HL_ParseEntities()` to expose entity data for subsequent conversion logic
  - `HL_WriteBSPFile(filename)` (if outputting to HL format) or defers to other format writers
  - `HL_FreeMaxBSP()` at shutdown
- **Shared entity interface** (`l_bsp_ent.h`):
  - `HL_ParseEntities()` populates the global `entities[]` array and `num_entities` count
  - `HL_UnparseEntities()` serializes back from `entities[]`
  - This allows any BSPC conversion module to work with entities uniformly, regardless of source format

### Outgoing (subsystem dependencies)
- **l_mem.h** (`GetMemory`, `FreeMemory`, `PrintMemorySize`):
  - Allocates all BSP lumps in a single batch via `HL_AllocMaxBSP`
  - Typical worst-case: ~MB-scale heap allocations; custom allocator avoids fragmentation
- **l_cmd.h** (file I/O, byte order):
  - `LoadFile()`: loads entire BSP into temp buffer
  - `SafeOpenWrite()`, `SafeWrite()`: writes BSP output
  - `LittleLong/Short/Float()`: endianness conversion (PC↔little-endian)
- **l_bsp_ent.h** (entity system):
  - `ParseEntity()`, `entities[]`, `num_entities` (extern globals, shared across all format handlers)
- **botlib/l_script.h** (entity string parsing):
  - `LoadScriptMemory()`, `SetScriptFlags()`, `FreeScript()`
  - Parses `hl_dentdata` (entity string lump) using botlib's C-like tokenizer
  - Allows text-based entity manipulation without format-specific parsing logic
- **l_log.h** (`Log_Print`)
  - Diagnostic logging of allocation/deallocation sizes

## Design Patterns & Rationale

### 1. **Global State Model (Era-Appropriate)**
   - Every HL BSP lump type has a global count (`hl_numX`) and pointer (`hl_dX`), plus checksum
   - Rationale: Pre-1990s game engine pattern. Simplifies stack-based code, guarantees single instance per process, avoids dynamic allocation overhead
   - Trade-off: Not reentrant, not thread-safe, but BSPC is single-threaded tool code

### 2. **Allocate-Max Strategy**
   - `HL_AllocMaxBSP()` pre-allocates `HL_MAX_MAP_*` slots for every lump type upfront
   - Rationale: HL map sizes are bounded by engine constants; pre-allocating avoids runtime reallocation during load/parse cycles
   - Trade-off: Wastes heap for small maps; tool-time OK since tool exits after one map

### 3. **Byte Swapping as I/O Middleware**
   - `HL_SwapBSPFile(todisk)` is called:
     - After load with `false`: little-endian → native-endian (for in-memory processing)
     - Before write with `true`: native-endian → little-endian (for disk output)
   - Rationale: Decouples geometry processing from endianness concerns; all in-memory ops assume native byte order
   - Implementation: Parameterized direction avoids code duplication

### 4. **Checksum-Based Change Detection**
   - Each lump computed via `FastChecksum()` after load; stored in `hl_*_checksum` globals
   - Rationale: Allows BSPC to detect which lumps were modified and only write changed ones (though not observed in `HL_WriteBSPFile` which writes all)

### 5. **Entity String Bridge**
   - Entities stored as raw `char *` in HL lumps, parsed to shared `entities[]` array
   - Rationale: Decouples entity logic from BSP format—conversion tools can manipulate entities uniformly regardless of source format (Q3, HL, Q1, etc.)
   - Implementation: Uses botlib's generic script parser to avoid HL-specific entity string grammar

### 6. **Run-Length Encoding for PVS**
   - `HL_CompressVis` / `HL_DecompressVis`: compress/decompress PVS rows using RLE
   - Rationale: HL visibility data compresses well (many zero bytes = invisible leaf ranges); RLE is simple and effective for sparse data
   - Applied only at load/write boundaries; in-memory PVS is stored uncompressed

## Data Flow Through This File

```
HL_LoadBSPFile(filename)
  ├─ LoadFile(filename) → entire BSP buffer
  ├─ Parse header (hl_header->lumps[])
  ├─ For each lump type:
  │   ├─ HL_CopyLump(i, dest, elemsize, maxcount)
  │   │   └─ memcpy from buffer → global array (with bounds check)
  │   └─ FastChecksum() → hl_*_checksum
  ├─ HL_SwapBSPFile(false) → little-endian to native byte order
  ├─ HL_DecompressVis() → expand PVS rows
  └─ HL_ParseEntities() → populate entities[]

Processing phase (external code mutates global arrays)

HL_WriteBSPFile(filename)
  ├─ HL_SwapBSPFile(true) → native to little-endian
  ├─ SafeOpenWrite() → output file handle
  ├─ For each lump:
  │   └─ HL_AddLump(i, data, len) → write + record offset/length in header
  ├─ HL_CompressVis() → compress PVS rows before writing
  ├─ fseek(0) + SafeWrite(header) → rewind and finalize header
  └─ close()
```

## Learning Notes

### Idiomatic Patterns from Q3A Era (~1999)
- **Global state dominance**: No encapsulation; all data is file-scope or global. Modern engines use objects/handles.
- **Worst-case allocation**: Pre-allocate for max size rather than grow dynamically. Reflects memory constraints of late-1990s systems.
- **Format-specific adapters**: BSPC uses separate `.c` files per BSP format rather than a plugin or factory pattern. Added entry points manually for each format.
- **Checksum-based validation**: Used before crypto hashes; `FastChecksum` is deterministic but not collision-resistant.
- **Direct struct mapping**: On-disk structs (`hl_dmodel_t`, `hl_dface_t`, etc.) mapped directly to memory, byte-swapped in-place. Modern engines serialize via explicit field-by-field code.

### Cross-Cutting Patterns
1. **Format abstraction layer**: All five `l_bsp_*.c` handlers expose `AllocMax`, `Free`, `Load`, `Write`, `ParseEntities`, `UnparseEntities`. BSPC dispatcher could polymorphically select the handler.
2. **Shared entity representation**: Entity manipulation is **format-independent** because `l_bsp_ent.h` provides a canonical `entities[]` array. Conversion between Q3↔HL↔Q1 is possible without format-specific logic.
3. **Tool vs. Runtime split**: This entire `bspc/` directory is **never linked into the game runtime**. At runtime, the game uses Q3 BSP format only (loaded in `code/qcommon/cm_load.c`). BSPC is a separate offline tool.

### Contrast with Runtime
- Runtime Q3 BSP loader (`code/qcommon/cm_load.c`) uses **not** global arrays but **heap-allocated structures** within a collision model context (`cmodel_t`), allowing multiple maps in memory (for per-player local collision models, etc.)
- Runtime loader does **not** deallocate per-lump; uses hunk-based lifetime management
- BSPC uses simpler allocate-all-at-once model because it processes one map at a time

## Potential Issues

### 1. **Critical Bug: Line ~142 (Edge Allocation)**
```c
hl_allocatedbspmem += HL_MAX_MAP_EDGES, sizeof(hl_dedge_t);  // WRONG
```
Should be `*` not `,`. The comma operator evaluates left-to-right and discards the left result, so `hl_allocatedbspmem` is incremented by the **value of `sizeof(hl_dedge_t)`** (a small constant like 4), not by the total edge buffer size. This causes:
- Wildly inaccurate memory accounting
- Misleading `Log_Print("allocated ... of BSP memory")` output
- Silent overflow if actual allocation exceeds reported total

### 2. **Bug: Line ~536 (Texdata Checksum)**
Per first-pass doc: `hl_dtexdata_checksum` is computed using `hl_numedges` (edge count) as the byte count instead of `hl_texdatasize`. This causes checksum of wrong data range.

### 3. **Implicit Dependency Order**
`HL_ParseEntities()` calls `LoadScriptMemory(hl_dentdata, hl_entdatasize)`. If `HL_UnparseEntities()` grows `hl_dentdata` but doesn't update `hl_entdatasize` correctly, subsequent re-parse fails silently or reads stale data.

---

These insights reveal **l_bsp_hl.c** as a straightforward format adapter serving BSPC's multi-format pipeline, with minimal cross-file dependencies (memory, I/O, entity abstraction) and simple state management optimized for offline tool usage.
