# code/botlib/be_aas_file.c — Enhanced Analysis

## Architectural Role

This file forms the **persistence layer for the botlib's navigation world**. It bridges offline map preprocessing (via `bspc`) and runtime pathfinding by serializing the `aasworld` global singleton to/from disk in a lump-based binary format. During engine startup, `AAS_LoadAASFile` is called by `be_aas_main.c:AAS_LoadMap` to populate the in-memory navigation graph; during shutdown or map change, `AAS_DumpAASData` tears down all allocations. The file acts as a checkpoint mechanism: once computed offline, the expensive AAS data (reachability links, cluster hierarchies, routing caches) is cached on disk and validated against the BSP checksum to ensure map coherency.

## Key Cross-References

### Incoming (who depends on this file)
- **`be_aas_main.c:AAS_LoadMap`** — calls `AAS_LoadAASFile` to load the navigation world during map initialization
- **`be_aas_main.c`** (initialization/shutdown) — calls `AAS_DumpAASData` to clean up stale data
- **`code/bspc/aas_file.c`** — offline tool that also implements `AAS_LoadAASFile`, `AAS_WriteAASFile`, `AAS_SwapAASData` for the map compiler
- **`be_aas_main.c:AAS_Setup`** — likely orchestrates the load sequence

### Outgoing (what this file depends on)
- **`botimport` vtable** (from `be_interface.h`) — all file I/O and memory operations (`FS_FOpenFile`, `FS_Read`, `FS_Write`, `FS_Seek`, `FreeMemory`)
- **`l_libvar.h:LibVarGetString`** — retrieves `sv_mapChecksum` cvar for BSP validation
- **`q_shared.h` endianness utilities** — `LittleLong`, `LittleFloat`, `LittleShort` for byte-swapping
- **`l_memory.h`** — `GetClearedHunkMemory`, `FreeMemory` for botlib's hunk-based allocation
- **`be_aas_main.h:AAS_Error`** — error reporting and recovery (calls `AAS_DumpAASData` on failure)

## Design Patterns & Rationale

**Lump-based format**: The AAS file mirrors the BSP structure: a fixed header followed by variable-length lumps indexed by `AASLUMP_*` constants. This design allows:
- Sparse files (empty lumps allocate dummy buffers)
- Easy offline tool integration (bspc uses the same serialization code)
- Forward compatibility (new lump types can be added without breaking readers)

**Byte-swap-on-load/write**: All integer and float fields are swapped using `LittleLong/Float/Short` to convert between file (little-endian) and native byte order. This is a classic portable binary format idiom, but **fragile**: double-swapping without reloading corrupts the data. The code mitigates this by warning in the `AAS_WriteAASFile` comment and immediately discarding the pointer after writing.

**XOR obfuscation**: The header (bytes 8+) of `AASVERSION` files are XOR'd with `data[i] ^= (unsigned char)(i * 119)`. This is **not cryptographic** but prevents casual file editing. The cipher is self-inverse (two applications = identity), making it suitable for toggling obfuscation during I/O.

**Checksum validation**: The BSP checksum embedded in the AAS file is validated against `sv_mapChecksum` to ensure the navigation data matches the loaded map geometry. Mismatch returns `BLERR_WRONGAASFILEVERSION` (arguably misleading nomenclature—should be `BLERR_AASOUTOFDATE`).

## Data Flow Through This File

**Load flow**:
1. Engine loads BSP, sets `sv_mapChecksum` cvar
2. `AAS_LoadAASFile(filename)` is called with map's `.aas` file path
3. `AAS_DumpAASData()` flushes any prior world
4. File header is read, identity/version/checksum validated
5. Each lump is sequentially read into hunk memory via `AAS_LoadAASLump`
6. `AAS_SwapAASData()` byte-swaps all lumps in-place (native byte order)
7. `aasworld.loaded = qtrue` signals readiness to routing/pathfinding layers

**Write flow** (offline tools only):
1. `AAS_WriteAASFile()` byte-swaps all lumps (file byte order)—**data is now corrupted**
2. Header written with placeholder offsets
3. Each lump written sequentially; offsets/lengths recorded in header
4. Header rewritten with final offsets
5. File closed; caller must not use `aasworld` further

## Learning Notes

- **Idiomatic endianness handling**: This demonstrates the classic mid-2000s approach: explicit byte-swap before/after serialization. Modern engines often use format metadata or platform-independent serialization (e.g., JSON, Protocol Buffers).
- **Hunk memory lifetime**: All AAS data is allocated from the "hunk"—a monotonically allocated region that is flushed wholesale on map change. No per-lump deallocation occurs; `AAS_DumpAASData` simply resets pointers and counts.
- **Weak obfuscation pattern**: The XOR cipher is deterministic and trivially reversed. It serves as a "don't edit by hand" signal rather than security.
- **Cross-compilation compatibility**: The botlib I/O is entirely decoupled from the engine's `qcommon` filesystem via the `botimport` vtable, allowing botlib to be compiled standalone for offline tools (`bspc`).

## Potential Issues

- **Double-free risk in `AAS_DumpAASData`**: `aasworld.numportals` is zeroed twice; `aasworld.portalindexsize` is not zeroed before freeing `portalindex`. Harmless here but inconsistent cleanup.
- **Byte-swap fragility**: If `AAS_SwapAASData` is accidentally called twice (e.g., due to control flow bug), data is silently corrupted until reload. No guard against this.
- **Confusing error code**: Checksum mismatch returns `BLERR_WRONGAASFILEVERSION`, which sounds like a version mismatch but actually signals map/AAS desynchronization.
- **Sequential read assumption**: `AAS_LoadAASLump` warns if lumps are read out-of-order (may indicate file fragmentation on disk), but still functions. This warning is informational only and never escalates to an error.
