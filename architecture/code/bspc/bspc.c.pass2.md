# code/bspc/bspc.c — Enhanced Analysis

## Architectural Role

This file serves as the **command-dispatch nexus and global configuration hub** for an offline standalone compiler tool that is completely decoupled from the runtime engine. Unlike the runtime code in `code/server`, `code/client`, and `code/cgame`, this tool runs once per invocation and owns no frame-loop lifecycle. It bridges three compile-time subsystems (`botlib` AAS, `bspc/*` geometry pipeline, and map I/O) by exposing them through a single argument-driven dispatch gate in `main()`.

## Key Cross-References

### Incoming (who depends on this file)
- **Platform layer** (`win32/`, `unix/`, `macosx/`) — provides `Sys_*`, `I_FloatTime()` clock, threading defaults
- **Logging** — `l_log.c` opens/closes via `Log_Open`/`Log_Close` called from `main()`
- **File enumeration** — `FindQuakeFiles()` (from `common/`) resolves glob and pak3-archive paths; used in `GetArgumentFiles()` and `CreateAASFilesForAllBSPFiles()`

### Outgoing (what this file depends on)
- **Geometry pipeline** (`code/bspc/aas_*.c`, `be_aas_bspc.c`) — primary clients of globals like `optimize`, `capsule_collision`, `create_aas`; these flags gate which compile steps execute
- **botlib/be_aas_cluster.h**, **botlib/be_aas_optimize.h** — called directly via `AAS_InitClustering`, `AAS_Optimize` to process navigation geometry
- **BSP/AAS I/O** (`aas_file.h`, `aas_cfg.h`) — `AAS_LoadAASFile`, `AAS_WriteAASFile` dispatched in case handlers
- **Map loading** — `LoadMapFromBSP()` called from `COMP_BSP2AAS` case; reads BSP, populates in-memory map
- **Configuration** — `LoadCfgFile()` early in `main()` to populate `aassettings` before any compile operation

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Global Configuration Hub** | ~25 `qboolean` and `float` globals + `aassettings_t` | Offline tools often centralize mutable compiler flags; reduces parameter threading through the 10+ module call chain |
| **Command-Line Dispatch** | `main()` loops args, sets flags in map table, then `switch(comp)` to six cases | Single-entry architecture is simpler than spawn-per-mode; allows shared config setup before branching |
| **Adapter/Bridge** | `be_aas_bspc.c` (not shown here but referenced) adapts `botlib_import_t` to compile-time file/memory services | Allows offline compiler to reuse botlib AAS pipeline without linking game engine; isolates dependency |
| **Graceful Degradation** | `onlyents` flag replaces entire map processing with entity-only reload loop | Common in tools: supports incremental workflow (re-assign entities without full rebuild) |

## Data Flow Through This File

```
Input → Argument Parsing (main)
  ↓
Load Configuration (DefaultCfg, LoadCfgFile)
  ↓
Select Compile Mode (comp = COMP_BSP2AAS, COMP_REACH, etc.)
  ↓
Dispatch to Pipeline
  ├─ BSP→AAS: LoadMapFromBSP → AAS_Create → AAS_InitClustering 
  │            → AAS_CalcReachAndClusters → (optional) AAS_Optimize 
  │            → AAS_WriteAASFile
  ├─ BSP→MAP: LoadMapFromBSP → WriteMapFile
  ├─ Reachability only: AAS_ContinueInitReachability
  ├─ Clustering: AAS_InitClustering
  ├─ Optimization: AAS_Optimize (in-place)
  └─ Info dump: AAS_LoadAASFile → AAS_ShowTotals
  ↓
Output (.aas, .map, or logs)
```

**Global flags act as compile-time gates:** `optimize` skips `AAS_Optimize`, `freetree` gates memory deallocation, `capsule_collision` changes collision model passed to AAS pipeline.

## Learning Notes

- **Old vs. New Pipeline:** Commented-out `ProcessWorldModel`/`ProcessSubModel`/`Map2Bsp` functions represent a Q2-era BSP-→-entity pipeline that was superseded by the `LoadMapFromBSP` + `AAS_Create` architecture. This is instructive: developers can see how the tool evolved from inline BSP processing to modular AAS compilation.

- **Platform Abstraction Limits:** `CreateAASFilesForAllBSPFiles()` shows platform-specific file enumeration (Win32 `FindFirstFile` vs. POSIX `glob`), a common pre-C11 pattern. Modern tools use platform-agnostic libraries, but this reveals Q3A's era (late 1990s–early 2000s).

- **Offline vs. Runtime Isolation:** This file demonstrates complete decoupling from game VMs (`code/cgame`, `code/game`, `code/ui`). The compiler **never touches** player state, snapshots, or client input—a crisp architectural boundary absent in integrated editors.

- **Configuration Cascading:** Config file values populate `aassettings` early, then individual CLI flags override them. This "file → CLI" precedence is idiomatic for offline tools but inverted from some game engines.

## Potential Issues

- **Global State Reentrance:** ~25 global flags + `aassettings` make the module non-reentrant. A hypothetical batch compiler or daemon calling `main()` multiple times would encounter stale globals. Mitigated by the tool's single-invocation design.

- **Fixed Buffer Limits:** `source[1024]`, `name[1024]`, `outbase[32]` assume path lengths ≤ 1024. On modern systems with longer paths, these overflow silently (strcpy has no bounds checking visible here). `AASOuputFile()` uses `MAX_PATH` elsewhere, suggesting this was an oversight.

- **Error Handling Severity:** `main()` calls `exit(0)` on parse failure, which aborts without cleanup. In contrast, modern tools use return codes and let OS/caller handle cleanup. Not a functional bug, but harsh for tools integrated into build pipelines.

- **Incomplete Batch Logic:** `CreateAASFilesForAllBSPFiles()` walks the file tree and logs matches but **does not trigger conversions**—it is dead code. The intent (batch processing all maps) was never implemented, suggesting the tool was primarily used interactively or via external scripts.
