# code/bspc/aas_file.h — Enhanced Analysis

## Architectural Role

This header bridges **offline AAS compilation** (in BSPC) with **runtime bot navigation** (in botlib). It declares the serialization contract for the Area Awareness System—a precomputed spatial partitioning and reachability graph consumed by the bot AI. While BSPC generates `.aas` files, botlib loads them at runtime; the header provides a unified interface across both codebases, enabling the tool→engine data flow that powers bot pathfinding.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/aas_file.c`** — BSPC offline compilation; implements `AAS_WriteAASFile` to serialize the constructed AAS world at the end of map compilation
- **`code/botlib/be_aas_file.c`** — Runtime bot library; implements `AAS_LoadAASFile` to deserialize pre-compiled AAS data during engine initialization
- **`code/bspc/bspc.c` / AAS compilation pipeline** — Calls `AAS_WriteAASFile` after `AAS_Create`, `AAS_OptimizeAll`, etc. complete
- **`code/botlib/be_aas_main.c` (`AAS_LoadMap`/`AAS_LoadFiles`)** — Calls `AAS_LoadAASFile` during bot library setup when loading a new map
- **`code/server/sv_bot.c`** — Indirectly, through botlib's public API (`BotLibSetup`)

### Outgoing (what this file depends on)
- **`code/bspc/aas_file.c` & `code/botlib/be_aas_file.c`** — Implementations of both functions (separate codebases, shared header)
- **`code/bspc/aas_store.h` / `code/botlib/be_aas_def.h`** — AAS data structure definitions (`aasworld_t`, lumps, area/face/edge arrays)
- **Platform filesystem** — File I/O primitives (via `botimport_t.FS_*` in botlib; native I/O in BSPC)
- **`qboolean` type** — From `q_shared.h`; both BSPC and botlib use this return convention

## Design Patterns & Rationale

**Dual-Implementation Pattern**: The same header is exposed by two separate implementations:
- **BSPC** (offline): builds in-memory AAS, optimizes geometry, serializes to disk
- **botlib** (runtime): deserializes from disk, populates the global `aasworld` singleton, validates integrity

This split exists because BSPC is a standalone tool with direct filesystem access and no VM/engine dependencies, while botlib runs inside the game engine and uses engine-provided I/O via `botlib_import_t.FS_*`.

**Archive-Embedded Data**: The `fpoffset` and `fplength` parameters enable reading AAS data from inside a larger file container (e.g., a `.pk3` ZIP archive), which is id Tech 3's standard asset packaging. This avoids requiring separate `.aas` files on disk in production deployments.

**Binary Serialization**: No text format; the `.aas` file is a direct memory dump of the `aasworld_t` structure, with byte-order swapping (`AAS_SwapAASData`) for cross-platform portability.

## Data Flow Through This File

```
Offline (BSPC Tool):
  BSP map → AAS construction (areas, faces, edges, clusters, reachability)
    → AAS_WriteAASFile("maps/q3dm1.aas")
      → .aas file on disk (or embedded in .pk3)

Runtime (Engine + botlib):
  Engine loads map
    → botlib.AAS_LoadMap("q3dm1") 
      → AAS_LoadAASFile("maps/q3dm1.aas", 0, 0)
        → populate aasworld_t in memory
        → AAS_InitRouting, AAS_InitClustering
  Bots use AAS for pathfinding (AAS_AreaRouteToGoalArea, etc.)
```

The `fpoffset`/`fplength` parameters allow `.aas` lumps to be stored inside a `.pk3` file at an arbitrary byte offset.

## Learning Notes

**Offline/Online Split**: Game engines typically separate tool-generated data (AAS) from runtime consumption. BSPC is a **pure offline tool** (no runtime role); botlib is **pure runtime** (no tool role). This header is the contract between them.

**Precomputed Navigation**: Unlike modern engines that compute navigation at runtime (dynamic graphs, Recast/Detour), id Tech 3 uses **static precomputed reachability**—14+ travel types (walk, jump, ladder, teleport, jump pad, etc.) precomputed for every area pair during BSPC compilation. This is fast at runtime but inflexible post-ship.

**Binary Recipes**: The `.aas` file is a "recipe"—compiled, serialized data that the engine consumes without recomputation. This is common across game engines for performance (BSP, lightmaps, AAS, etc.).

**Portable Binary Format**: The implementation uses byte-swapping (`AAS_SwapAASData`) to handle big-endian/little-endian differences across platforms, a pattern born from id Tech 3's multi-platform deployment (x86, PPC, MIPS).

## Potential Issues

**Limited Error Recovery**: Both functions return only `qboolean` (success/failure); no error codes or context. Callers cannot distinguish between "file not found," "corrupted data," "version mismatch," or other failure modes. Modern engines typically serialize version numbers and CRCs into binary formats.

**No Validation on Load**: The header does not show bounds checks or integrity validation within the function signatures; implementation (`be_aas_file.c`) must handle this internally.

**Tight Coupling Between Tool & Runtime**: If BSPC changes the `.aas` format, all existing `.aas` files become incompatible with no versioning mechanism visible here.
