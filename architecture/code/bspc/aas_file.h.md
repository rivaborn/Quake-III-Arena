# code/bspc/aas_file.h

## File Purpose
Public interface header for AAS (Area Awareness System) file I/O operations within the BSPC (BSP Compiler) tool. It exposes exactly two functions for writing and loading compiled AAS navigation data to and from disk.

## Core Responsibilities
- Declare the AAS file write entry point for serializing compiled navigation data
- Declare the AAS file load entry point for deserializing navigation data with optional subrange support

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_WriteAASFile
- **Signature:** `qboolean AAS_WriteAASFile(char *filename)`
- **Purpose:** Serializes the current in-memory AAS world model to a binary `.aas` file on disk.
- **Inputs:** `filename` — path/name of the output AAS file to write.
- **Outputs/Return:** `qboolean` — `qtrue` on success, `qfalse` on failure.
- **Side effects:** File I/O (creates/overwrites file on disk).
- **Calls:** Defined in `aas_file.c`; callers not inferable from this header alone.
- **Notes:** Used by the BSPC tool after AAS construction is complete.

### AAS_LoadAASFile
- **Signature:** `qboolean AAS_LoadAASFile(char *filename, int fpoffset, int fplength)`
- **Purpose:** Deserializes an AAS file from disk into memory, with support for reading a subrange of the file (e.g., from a packed archive).
- **Inputs:** `filename` — path to the AAS file; `fpoffset` — byte offset into the file to begin reading; `fplength` — number of bytes to read (0 likely means full file).
- **Outputs/Return:** `qboolean` — `qtrue` on success, `qfalse` on failure.
- **Side effects:** Memory allocation; populates global AAS world state defined in `aas_file.c` / `aas_store.c`.
- **Calls:** Defined in `aas_file.c`; callers not inferable from this header alone.
- **Notes:** `fpoffset`/`fplength` parameters suggest `.aas` data may be embedded inside a larger container file (e.g., a PK3/PAK).

## Control Flow Notes
This header is consumed during the BSPC compilation pipeline. `AAS_WriteAASFile` is called at the end of AAS generation to persist results. `AAS_LoadAASFile` may be called to reload or verify a previously compiled AAS file. Neither function participates in a frame/update loop — both are offline tool operations.

## External Dependencies
- `qboolean` — defined in `q_shared.h` (or equivalent Quake shared header); used as the boolean return type.
- Implementation resides in `code/bspc/aas_file.c`.
- AAS world model state populated/consumed by this module is defined elsewhere (`aas_store.h`, `aas_create.h`).
