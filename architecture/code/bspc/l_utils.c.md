# code/bspc/l_utils.c

## File Purpose
Provides cross-platform filesystem path utility functions for both the BSPC map compiler and the BOTLIB bot library. It normalizes path separators and ensures paths are properly terminated, with a conditionally compiled vector-to-angles conversion for BOTLIB use.

## Core Responsibilities
- Convert direction vectors to Euler angles (BOTLIB only)
- Normalize filesystem path separator characters to the platform-appropriate character
- Append a trailing path separator to directory strings safely
- Provide disabled (guarded with `#if 0`) legacy Quake 2 PAK file search routines

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### Vector2Angles *(BOTLIB only)*
- **Signature:** `void Vector2Angles(vec3_t value1, vec3_t angles)`
- **Purpose:** Converts a 3D direction vector into Pitch/Yaw/Roll Euler angles.
- **Inputs:** `value1` — normalized direction vector
- **Outputs/Return:** `angles` — filled with `[PITCH, YAW, ROLL]` in degrees; ROLL is always 0
- **Side effects:** None
- **Calls:** `atan2`, `sqrt` (standard C math)
- **Notes:** Handles the degenerate case where X and Y components are both zero (vertical vector). Pitch is negated on output. Only compiled when `BOTLIB` is defined.

### ConvertPath
- **Signature:** `void ConvertPath(char *path)`
- **Purpose:** Walks a C string in-place and replaces all `/` and `\` characters with the platform path separator (`PATHSEPERATOR_CHAR`).
- **Inputs:** `path` — mutable path string
- **Outputs/Return:** void; modifies `path` in place
- **Side effects:** Mutates the input buffer
- **Calls:** None
- **Notes:** `PATHSEPERATOR_CHAR` is defined externally per platform (e.g., `'/'` on Linux, `'\\'` on Windows).

### AppendPathSeperator
- **Signature:** `void AppendPathSeperator(char *path, int length)`
- **Purpose:** Appends a platform path separator to `path` if it does not already end with one and the buffer has room.
- **Inputs:** `path` — directory string; `length` — total buffer capacity
- **Outputs/Return:** void; modifies `path` in place
- **Side effects:** Mutates the input buffer
- **Calls:** `strlen`
- **Notes:** Guards against empty strings and buffer overflow. The function name in the closing comment contains a typo (`AppenPathSeperator`).

### FindFileInPak / FindQuakeFile2 / FindQuakeFile *(disabled)*
- All enclosed in `#if 0` and not compiled.
- **Notes:** Legacy Quake 2 PAK archive search routines. Would open a `.pak` file, parse its directory, and locate a named file by offset/length. `FindQuakeFile` had a BOTLIB vs. non-BOTLIB split for how `basedir`/`gamedir` are supplied.

## Control Flow Notes
This file provides only utility/helper functions. It has no frame or init/shutdown hooks. `ConvertPath` and `AppendPathSeperator` are called wherever file paths are constructed in the BSPC tool (e.g., in `l_qfiles.c`, map loaders). `Vector2Angles` is called from BOTLIB movement or debug code.

## External Dependencies
- **BOTLIB build path:** `q_shared.h`, `qfiles.h`, `botlib.h`, `l_log.h`, `l_libvar.h`, `l_memory.h`, `be_interface.h`
- **BSPC build path:** `qbsp.h`, `l_mem.h`
- **`PATHSEPERATOR_CHAR`** — defined elsewhere (platform headers or `qbsp.h`/`l_utils.h`)
- **`M_PI`, `atan2`, `sqrt`** — standard C `<math.h>`
- **`PITCH`, `YAW`, `ROLL`** — index constants from `q_shared.h`
- **`Log_Write`** — declared in `l_log.h`; used only in the disabled `#if 0` block
- **`LibVarGetString`** — declared in `l_libvar.h`; used only in the disabled block
