# code/botlib/be_aas_file.h

## File Purpose
Header file declaring internal AAS (Area Awareness System) file I/O operations for the Quake III bot library. All declarations are gated behind the `AASINTERN` preprocessor guard, restricting visibility to internal botlib compilation units only.

## Core Responsibilities
- Declare the AAS file load interface
- Declare the AAS file write interface
- Declare AAS data cleanup and diagnostic utilities
- Enforce internal-only access via `AASINTERN` guard

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_LoadAASFile
- **Signature:** `int AAS_LoadAASFile(char *filename)`
- **Purpose:** Loads a compiled AAS navigation file from disk into the bot library's internal AAS world state.
- **Inputs:** `filename` — path/name of the `.aas` file to load.
- **Outputs/Return:** `int` — likely a success/error code (1 = success, 0 = failure, by Q3 convention).
- **Side effects:** Populates global AAS world data structures (defined in `be_aas_def.h`/`be_aas_main.c`).
- **Calls:** Not inferable from this file.
- **Notes:** Implementation resides in `be_aas_file.c`.

### AAS_WriteAASFile
- **Signature:** `qboolean AAS_WriteAASFile(char *filename)`
- **Purpose:** Serializes the current in-memory AAS data back to a file on disk.
- **Inputs:** `filename` — output file path.
- **Outputs/Return:** `qboolean` — `qtrue` on success, `qfalse` on failure.
- **Side effects:** File I/O write; no in-memory state changes expected.
- **Calls:** Not inferable from this file.
- **Notes:** Used by the BSPC tool during AAS compilation, not at game runtime.

### AAS_DumpAASData
- **Signature:** `void AAS_DumpAASData(void)`
- **Purpose:** Frees or clears all loaded AAS data from memory.
- **Inputs:** None.
- **Outputs/Return:** `void`.
- **Side effects:** Releases global AAS world state; inverse of `AAS_LoadAASFile`.
- **Calls:** Not inferable from this file.
- **Notes:** Called on map unload or botlib shutdown.

### AAS_FileInfo
- **Signature:** `void AAS_FileInfo(void)`
- **Purpose:** Prints diagnostic information about the currently loaded AAS file (area counts, reachability counts, etc.).
- **Inputs:** None.
- **Outputs/Return:** `void`.
- **Side effects:** Output to bot library log/console.
- **Calls:** Not inferable from this file.
- **Notes:** Developer/debug utility.

## Control Flow Notes
These functions are invoked during botlib initialization (`AAS_LoadAASFile`) when a map loads, during shutdown (`AAS_DumpAASData`), and optionally during BSPC tool compilation (`AAS_WriteAASFile`). `AAS_FileInfo` is a debug-time call with no frame-loop involvement.

## External Dependencies
- `AASINTERN` macro — must be defined by the including translation unit to expose these declarations.
- `qboolean` — defined in `q_shared.h` (engine shared types).
- Implementation: `code/botlib/be_aas_file.c`
