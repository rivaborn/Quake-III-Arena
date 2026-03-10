# code/bspc/bspc.c

## File Purpose
This is the main entry point and global state hub for the BSPC (BSP Compiler) tool, a standalone offline utility that converts Quake BSP files into AAS (Area Awareness System) navigation files consumed by the bot AI. It parses command-line arguments and dispatches to the appropriate conversion pipeline.

## Core Responsibilities
- Define and own all global BSP/AAS compilation flags (nocsg, optimize, freetree, etc.)
- Parse `main()` command-line arguments and map them to compilation modes
- Dispatch to one of six compilation operations: BSP→MAP, BSP→AAS, reachability, clustering, AAS optimization, AAS info
- Construct output `.aas` file paths from input file metadata
- Enumerate all BSP files under a Quake directory tree (Win32 and POSIX)
- Collect and resolve argument file lists via glob/pak-aware `FindQuakeFiles`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `quakefile_t` | struct (defined elsewhere) | Describes a located source file, including pak/pk3 offset, original name, and type |
| `aas_settings_t` | struct (defined elsewhere) | AAS generation configuration, stored as global `aassettings` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `subdivide_size` | `float` | global | BSP face subdivision size |
| `source` | `char[1024]` | global | Source file path stem |
| `name` | `char[1024]` | global | Full source map file name |
| `microvolume` | `vec_t` | global | Minimum brush volume threshold |
| `outbase` | `char[32]` | global | Output base directory |
| `entity_num` | `int` | global | Current entity index during BSP processing |
| `aassettings` | `aas_settings_t` | global | AAS generation settings |
| `noprune`, `glview`, `nodetail`, `fulldetail`, `onlyents`, `nomerge`, `nowater`, `nocsg`, `noweld`, `noshare`, `nosubdiv`, `notjunc` | `qboolean` | global | BSP pipeline feature flags |
| `optimize` | `qboolean` | global | Enable AAS post-optimization pass |
| `leaktest` | `qboolean` | global | Abort on leaked map |
| `verboseentities` | `qboolean` | global | Verbose entity logging |
| `freetree` | `qboolean` | global | Free BSP tree after AAS creation |
| `create_aas` | `qboolean` | global | Signal to map loader that AAS output is expected |
| `nobrushmerge` | `qboolean` | global | Disable brush merging pass |
| `lessbrushes` | `qboolean` | global | Prefer fewer brushes over correct UV placement |
| `cancelconversion` | `qboolean` | global | External cancellation flag |
| `noliquids` | `qboolean` | global | Omit liquid brushes from map output |
| `forcesidesvisible` | `qboolean` | global | Mark all BSP sides visible on load |
| `capsule_collision` | `qboolean` | global | Use capsule collision model |

## Key Functions / Methods

### AASOuputFile
- **Signature:** `void AASOuputFile(quakefile_t *qf, char *outputpath, char *filename)`
- **Purpose:** Derives the output `.aas` file path from the source `quakefile_t`. Uses `outputpath` if provided; otherwise places alongside the source file (or inside a `maps/` subfolder for pk3/pak/sin sources).
- **Inputs:** `qf` — source file descriptor; `outputpath` — optional override directory; `filename` — out buffer
- **Outputs/Return:** Fills `filename` with the resolved path; void return.
- **Side effects:** May call `CreatePath` to mkdir the `maps/` directory.
- **Calls:** `AppendPathSeperator`, `ExtractFileBase`, `ExtractFileExtension`, `access`, `CreatePath`
- **Notes:** Handles three cases: explicit output path, pk3/pak/sin source (writes into adjacent `maps/`), and plain file source (replaces extension).

### CreateAASFilesForAllBSPFiles
- **Signature:** `void CreateAASFilesForAllBSPFiles(char *quakepath)`
- **Purpose:** Walks a Quake installation root, finds all BSP and AAS files, and logs them for batch comparison. Currently only logs; it does not trigger conversions.
- **Inputs:** `quakepath` — root directory to scan
- **Outputs/Return:** void; logs found files.
- **Side effects:** File system enumeration (Win32 `FindFirstFile`/`FindNextFile` or POSIX `glob`/`globfree`).
- **Calls:** `FindQuakeFiles`, `Log_Print`, `AppendPathSeperator`

### GetArgumentFiles
- **Signature:** `quakefile_t *GetArgumentFiles(int argc, char *argv[], int *i, char *ext)`
- **Purpose:** Consumes consecutive non-flag arguments from `argv` starting at `(*i)+1`, normalizes their extensions to `ext`, resolves them via `FindQuakeFiles`, and returns a linked list.
- **Inputs:** argc/argv — CLI args; `i` — current index (advanced in place); `ext` — desired file extension
- **Outputs/Return:** Linked list of `quakefile_t *`, or NULL.
- **Side effects:** Advances `*i`.
- **Calls:** `FindQuakeFiles`

### main
- **Signature:** `int main(int argc, char **argv)`
- **Purpose:** Program entry point. Opens the log, loads default config, iterates all CLI arguments to set globals and select `comp` mode, then dispatches to the matching conversion case.
- **Inputs:** Standard argc/argv
- **Outputs/Return:** 0 on normal exit; `exit(0)` used internally on fatal errors.
- **Side effects:** Opens/closes `bspc.log`; sets numerous global flags; calls conversion pipeline functions that allocate/free large AAS data structures; calls `exit()` on config parse error.
- **Calls:** `Log_Open`, `Log_Print`, `Log_Close`, `DefaultCfg`, `I_FloatTime`, `LoadCfgFile`, `GetArgumentFiles`, `CreateAASFilesForAllBSPFiles`, `LoadMapFromBSP`, `WriteMapFile`, `AAS_Create`, `AAS_CalcReachAndClusters`, `AAS_Optimize`, `AAS_WriteAASFile`, `AAS_FreeMaxAAS`, `AAS_LoadAASFile`, `AAS_InitBotImport`, `AAS_InitClustering`, `AAS_ShowTotals`, `Warning`, `Error`

## Control Flow Notes
This file is the **offline tool entry point** — it has no game-engine frame loop. `main()` runs once: init → arg parse → single dispatch → exit. It does not participate in the runtime game init/frame/shutdown cycle. The commented-out `ProcessWorldModel`/`ProcessSubModel`/`ProcessModels`/`Map2Bsp` functions represent an older Q2-era pipeline that was superseded by the `LoadMapFromBSP` + `AAS_Create` approach.

## External Dependencies
- `qbsp.h` — BSP pipeline types, all map/brush/node/portal structs, and declarations for the majority of processing functions
- `l_mem.h` — memory allocation
- `botlib/aasfile.h` — AAS on-disk format constants and structs
- `botlib/be_aas_cluster.h` — `AAS_InitClustering`
- `botlib/be_aas_optimize.h` — `AAS_Optimize`
- `aas_create.h`, `aas_store.h`, `aas_file.h`, `aas_cfg.h` — AAS build/IO pipeline (defined elsewhere)
- `be_aas_bspc.h` — `AAS_CalcReachAndClusters`, `AAS_InitBotImport` (defined elsewhere)
- `use_nodequeue` — extern from `brushbsp.c`
- `calcgrapplereach` — extern from `be_aas_reach.c`
- POSIX: `unistd.h`, `glob.h`, `sys/stat.h`; Win32: `direct.h`, `windows.h`
