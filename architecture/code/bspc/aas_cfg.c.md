# code/bspc/aas_cfg.c

## File Purpose
Manages the AAS (Area Awareness System) configuration for the BSPC map compiler tool. It defines, loads, and applies physics and reachability settings used during AAS file generation from BSP maps.

## Core Responsibilities
- Define field descriptor tables (`fielddef_t`) and struct descriptors (`structdef_t`) for `cfg_t` and `aas_bbox_t` using offset macros
- Provide default Q3A configuration values via `DefaultCfg()`
- Parse a `.cfg` file using the botlib precompiler to populate the global `cfg` struct
- Validate loaded configuration (gravity direction magnitude, bounding box count)
- Propagate loaded float config values into the botlib libvar system via `SetCfgLibVars()`
- Provide a `va()` varargs string formatting utility

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `cfg_t` | struct (typedef) | Holds all physics/reachability parameters and bounding box definitions used during AAS generation |
| `aas_bbox_t` | struct (typedef) | Describes a single player bounding box (presence type, flags, mins/maxs) |
| `fielddef_t` | typedef (defined in `l_struct.h`) | Describes a struct field by name, offset, and type for generic config parsing |
| `structdef_t` | typedef (defined in `l_struct.h`) | Pairs a struct size with its `fielddef_t` array for generic parsing |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cfg` | `cfg_t` | global | Single global config instance consumed by all AAS generation code |
| `bbox_fields` | `fielddef_t[]` | file-static (file scope) | Field descriptors for `aas_bbox_t` |
| `cfg_fields` | `fielddef_t[]` | file-static (file scope) | Field descriptors for `cfg_t` |
| `bbox_struct` | `structdef_t` | file-static (file scope) | Struct descriptor for `aas_bbox_t` |
| `cfg_struct` | `structdef_t` | file-static (file scope) | Struct descriptor for `cfg_t` |
| `string[2]` (in `va`) | `char[2][32000]` | static local | Ping-pong buffer for varargs formatting |

## Key Functions / Methods

### DefaultCfg
- **Signature:** `void DefaultCfg(void)`
- **Purpose:** Initializes `cfg` to safe Q3A defaults; sets all float fields to `FLT_MAX` (sentinel for "not set"), then hard-codes two bounding boxes (standing and crouching) and basic gravity direction/steepness.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes to global `cfg`
- **Calls:** Iterates `cfg_fields` by index; no external function calls beyond direct struct assignment
- **Notes:** `FLT_MAX` is used as a sentinel meaning "unspecified"; `SetCfgLibVars` skips these when publishing to libvars.

### va
- **Signature:** `char * QDECL va(char *format, ...)`
- **Purpose:** Varargs sprintf into a static ping-pong buffer; used to format float values as strings for `LibVarSet`.
- **Inputs:** `format` — printf format string; variadic args
- **Outputs/Return:** Pointer to one of two static 32 KB buffers (alternates each call)
- **Side effects:** Writes to static local buffers; not thread-safe
- **Calls:** `vsprintf`
- **Notes:** Two-slot rotation allows one level of nested calls without clobbering; deeper nesting will corrupt output.

### SetCfgLibVars
- **Signature:** `void SetCfgLibVars(void)`
- **Purpose:** Iterates all `cfg_fields`; for each float field that is not `FLT_MAX`, calls `LibVarSet` to register it in the botlib variable system.
- **Inputs:** None (reads global `cfg`)
- **Outputs/Return:** void
- **Side effects:** Writes libvars via `LibVarSet`; those vars are consumed by the botlib at runtime.
- **Calls:** `LibVarSet`, `va`
- **Notes:** Only float fields are published; integer/array fields are not handled here.

### LoadCfgFile
- **Signature:** `int LoadCfgFile(char *filename)`
- **Purpose:** Opens and parses a text config file via the botlib precompiler. Reads `bbox` and `settings` blocks into the global `cfg`, validates the result, then calls `SetCfgLibVars`.
- **Inputs:** `filename` — path to `.cfg` file
- **Outputs/Return:** `true` on success, `false` on load or parse failure
- **Side effects:** Zeroes and repopulates global `cfg`; registers libvars; prints to log
- **Calls:** `LoadSourceFile`, `PC_ReadToken`, `stricmp`, `ReadStructure`, `FreeSource`, `VectorLength`, `SourceError`, `SourceWarning`, `SetCfgLibVars`, `Log_Print`
- **Notes:** Gravity direction is validated to unit length (0.9–1.1). At least one bbox must be defined. Duplicate `settings` blocks produce a warning but are allowed.

## Control Flow Notes
Called from the BSPC tool's initialization path before AAS generation begins. `DefaultCfg()` is called first to establish safe defaults; `LoadCfgFile()` is then called with a game-specific `.cfg` to override them. The resulting `cfg` global is read by other `aas_*` subsystems throughout the AAS build pipeline. No per-frame update cycle; this is purely a one-shot init.

## External Dependencies
- `qbsp.h` — BSPC-wide types and declarations
- `float.h` — `FLT_MAX`
- `../botlib/aasfile.h` — `aas_bbox_t`, presence type constants (`PRESENCE_NORMAL`, `PRESENCE_CROUCH`)
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t`, `cfg` extern declaration
- `../botlib/l_precomp.h` — `source_t`, `token_t`, `LoadSourceFile`, `FreeSource`, `PC_ReadToken`, `SourceError`, `SourceWarning`
- `../botlib/l_struct.h` — `fielddef_t`, `structdef_t`, `ReadStructure`, `FT_FLOAT`, `FT_INT`, `FT_ARRAY`, `FT_TYPE`
- `../botlib/l_libvar.h` — `LibVarSet` (defined elsewhere, in botlib)
- `VectorLength` — defined in math utility (botlib/game shared)
- `Log_Print` — defined in `l_log.c`
