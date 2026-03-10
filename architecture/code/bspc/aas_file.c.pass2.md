# code/bspc/aas_file.c — Enhanced Analysis

## Architectural Role

This file implements the serialization layer for the BSPC offline compiler's AAS (Area Awareness System) file format. It's the **final output stage** of map compilation: after `aas_create.c` populates `aasworld` with navigation geometry, this file persists it to disk in a binary lump format that the runtime botlib will later load. As a build-time tool, it bridges AAS generation (offline) with AAS consumption (runtime), meaning the versioning, byte-swapping, and obfuscation logic here must exactly match `code/botlib/be_aas_file.c` to ensure round-trip compatibility.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/be_aas_bspc.c`** — Calls `AAS_LoadAASFile()` and `AAS_WriteAASFile()` as part of the map compilation pipeline; also invokes `AAS_ShowTotals()` for diagnostic output during compilation.
- **BSPC main loop** (`bspc.c`) — Orchestrates the overall compilation; this file is invoked when the `-aas` flag is set, triggering full AAS generation.

### Outgoing (what this file depends on)
- **`aasworld` global** (from `aas_store.h`) — Central AAS world state; all 14 lumps are fields of this `aas_t` struct. This file reads/writes all of them.
- **`qbsp.h` utilities** — Memory allocation (`GetClearedMemory`, `FreeMemory`), byte-swap helpers (`LittleLong`, `LittleFloat`, `LittleShort`), and error handling (`Error` macro aliased to `AAS_Error`).
- **Standard C I/O** — `fopen`, `fseek`, `fread`, `fwrite`, `fclose`, `ftell`.
- **Logging** — `Log_Print()` from the BSPC logging subsystem (declared in `l_log.h`).
- **Endian type definitions** — `aas_header_t`, `aas_lump_t`, and all per-lump struct types from `code/botlib/aasfile.h`.

## Design Patterns & Rationale

### 1. **Versioned Binary Lump Format**
The AAS file uses a header-plus-14-lumps structure:
- **Header** contains magic ID (`AASID`), version (4 or 5), BSP checksum, and a 14-element lump table.
- **Lumps** store variable-length arrays: bounding boxes, vertices, planes, edges, faces, areas, clusters, etc.
- **Why this design?** Allows incremental streaming, partial reloading, and independent lump processing. Mirrors BSP lump architecture (`qbsp.h`).

### 2. **Symmetric Endian Byte-Swapping**
- On **load**: Read file (always little-endian), then `AAS_SwapAASData()` to host byte-order.
- On **write**: `AAS_SwapAASData()` (corrupt to little-endian), write file, reload is required to use data again.
- **Why?** This era (early 2000s) prioritized cross-platform support. Little-endian is the standard on x86, but big-endian targets (PowerPC, old Macs) needed swapping. The in-place swap avoids a separate temporary buffer.

### 3. **XOR Header Obfuscation (Version 5)**
- `AAS_DData()` applies `data[i] ^= (i * 119)` per byte to the header (bytes 8+) on both load and write.
- **Not cryptographic.** Just deters casual inspection of file structure.
- **Why version this?** Old AAS files (v4) don't use obfuscation; v5 added it. The tool must handle both for compatibility.

### 4. **Error Path Cascading**
- When `AAS_LoadAASLump()` fails, it calls `AAS_DumpAASData()` and `fclose()` itself.
- **Problem:** The caller (`AAS_LoadAASFile`) continues checking return values and will also call `AAS_DumpAASData()`, leading to redundant calls. Also, on partial load, the file is already closed, but the next lump load would try to seek on a closed handle.

## Data Flow Through This File

```
[BSPC Compilation Pipeline]
         ↓
   AAS_Create() → aasworld populated with arrays
         ↓
 AAS_WriteAASFile() ← User invokes save at end of map compile
    │
    ├─→ AAS_SwapAASData()  [corrupt in-memory data to file byte-order]
    ├─→ fopen() output file
    ├─→ Write header (with AASID, AASVERSION)
    ├─→ For each lump: AAS_WriteAASLump()
    │   ├─→ ftell() → record offset in header.lumps[i]
    │   └─→ fwrite() data
    ├─→ fseek() back to header
    ├─→ AAS_DData() on header (obfuscate)
    ├─→ fwrite() header
    └─→ fclose()
    
   [Result: Binary AAS file on disk, aasworld now unusable (byte-swapped)]
         ↓
 [At runtime, botlib will AAS_LoadAASFile() to read it back]
```

**Load flow (less common in BSPC, used for incremental editing):**
```
 AAS_LoadAASFile(filename)
    ├─→ AAS_DumpAASData()  [mark as unloaded]
    ├─→ fopen() input file
    ├─→ fread() & validate header
    ├─→ If version == 5: AAS_DData() to deobfuscate header
    ├─→ For each lump:
    │   └─→ AAS_LoadAASLump() [seek, alloc, read]
    ├─→ AAS_SwapAASData()  [convert from file byte-order to host]
    ├─→ aasworld.loaded = true
    └─→ fclose()
```

## Learning Notes

### What This File Teaches
1. **Binary I/O Best Practices (2000s era):**
   - Structured format versioning (AASVERSION_OLD vs AASVERSION).
   - Explicit byte-swapping for cross-platform data interchange.
   - Validation (magic ID, version check) before attempting parse.

2. **Lump Architecture:**
   - Used throughout id Tech 3 (BSP files, shader compilation, AAS).
   - Enables streaming and partial I/O; mirrors the BSP lump table pattern.

3. **Diagnostic Logging in Build Tools:**
   - `AAS_ShowTotals()` demonstrates how to output summary statistics at build time.
   - Travel-type breakdown (`TRAVEL_*` constants) shows how the tool understands game mechanics.

### Idiomatic Design for This Era / Engine
- **No C++ abstraction layers** — Direct struct manipulation, manual memory management.
- **Global state (`aasworld`)** — Simpler than passing structs but requires careful sequencing (swap must happen at the right time).
- **Byte-order as a build concern** — Modern engines often standardize on one endianness; this tool had to support multiple targets (x86, PPC, Alpha).
- **Obfuscation-lite** — XOR is not encryption; just a lightweight anti-tampering measure.

### Modern Engine Comparison
- **Modern engines** typically use a portable serialization format (JSON, protobuf, or binary with explicit endianness markers) rather than host-endian + symmetric swap.
- **Navigation mesh formats** (Recast/Detour, NavMeshAsset) use versioned lump-style layouts but with stricter bounds checking and streaming validation.
- **Tool-Runtime Parity:** This file shows how a build tool must mirror the runtime loader exactly. Modern practices (code generation, shared type definitions) reduce this duplication risk.

## Potential Issues

### Critical
1. **Unchecked Lump Offsets & Lengths**
   - `AAS_LoadAASLump()` performs no bounds validation on `offset` or `length`.
   - A corrupted header could cause `fseek()` to far-out file positions or read past EOF.
   - **Modern fix:** Validate all offsets against file size; use safe arithmetic.

2. **Memory Leak in AAS_DumpAASData**
   - The actual `FreeMemory()` calls are commented out (lines ~168–181).
   - If `AAS_LoadAASFile()` is called multiple times (e.g., to reload an existing AAS), the old arrays are orphaned.
   - **Impact:** In a long-running compiler or editor, this could accumulate memory waste.

### Moderate
3. **Shared Global State Fragility**
   - `aasworld` is a singleton. After `AAS_WriteAASFile()`, the data is byte-swapped and unusable; code must either reload or avoid using it.
   - No runtime checks to prevent accidental reads of corrupted data.
   - **Modern fix:** Encapsulate in a struct with explicit lifecycle guards (e.g., `aasworld.swapped` flag).

4. **Error Path File Handle Leaks**
   - If `AAS_LoadAASLump()` encounters `fread()` error, it closes the file.
   - But `AAS_LoadAASFile()` may also call `fclose(fp)` later, causing a double-close on some platforms (undefined behavior).
   - **Fix:** Return file-closed status via out parameter or move cleanup into the caller.

### Minor
5. **Log Message Format Inconsistency**
   - `AAS_WriteAASLump()` logs lump number as `%s` but passes `lumpnum` (an `int`).
   - This is a format string bug (should be `%d`), though it may not crash due to luck with stack layout.
