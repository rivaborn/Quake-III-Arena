# code/bspc/l_utils.h

## File Purpose
A utility header for the BSPC (BSP Compiler) tool, providing cross-platform path handling macros, math convenience macros, and declarations for file-finding utilities used during BSP/AAS compilation.

## Core Responsibilities
- Define cross-platform path separator macros (`\\` vs `/`)
- Provide math utility macros (random, clamp, abs, axis indices)
- Declare the `foundfile_t` structure for locating files inside pak archives
- Declare file-search functions for locating Quake assets on disk or in pak files
- Declare the `Vector2Angles` conversion utility

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `foundfile_t` | struct | Stores a located file's byte offset, length, and filename (used for pak file lookups) |

## Global / File-Static State
None.

## Key Functions / Methods

### Vector2Angles
- Signature: `void Vector2Angles(vec3_t value1, vec3_t angles)`
- Purpose: Converts a direction vector to Euler angles.
- Inputs: `value1` — direction vector; `angles` — output angle triple.
- Outputs/Return: void; result written to `angles`.
- Side effects: None inferable from declaration.
- Calls: Not inferable from this file.
- Notes: Declaration only; implementation is elsewhere.

### ConvertPath
- Signature: `void ConvertPath(char *path)`
- Purpose: Normalizes path separators in-place to the platform-correct character.
- Inputs: `path` — mutable C string.
- Outputs/Return: void; mutates `path` in place.
- Side effects: Modifies input buffer.
- Calls: Not inferable from this file.

### AppendPathSeperator
- Signature: `void AppendPathSeperator(char *path, int length)`
- Purpose: Appends a trailing path separator to `path` if it fits within `length`.
- Inputs: `path` — mutable string; `length` — buffer capacity.
- Outputs/Return: void; mutates `path`.
- Side effects: Modifies input buffer.
- Calls: Not inferable from this file.

### FindFileInPak
- Signature: `qboolean FindFileInPak(char *pakfile, char *filename, foundfile_t *file)`
- Purpose: Searches a specific pak archive for a named file and fills `file` with its location metadata.
- Inputs: `pakfile` — path to the pak; `filename` — asset name to find; `file` — output struct.
- Outputs/Return: `qboolean` — true if found.
- Side effects: Reads from disk; writes to `*file`.
- Calls: Not inferable from this file.

### FindQuakeFile
- Signature (BOTLIB): `qboolean FindQuakeFile(char *filename, foundfile_t *file)`
- Signature (non-BOTLIB): `qboolean FindQuakeFile(char *basedir, char *gamedir, char *filename, foundfile_t *file)`
- Purpose: Searches the Quake directory tree (and pak files) for a named asset; signature varies by compile target.
- Inputs: Optional `basedir`/`gamedir` for non-botlib builds; `filename`; output `file`.
- Outputs/Return: `qboolean` — true if found.
- Side effects: Reads from filesystem and/or pak files; writes to `*file`.
- Notes: The `#ifdef BOTLIB` guard selects between a two-argument and four-argument version, allowing shared logic between the standalone BSPC tool and the botlib runtime.

## Control Flow Notes
This is a header-only declaration file with no executable logic. It is included by BSPC compilation units that need path manipulation or Quake file location services at BSP/AAS build time, not at game runtime.

## External Dependencies
- `vec3_t` — defined in Quake shared math headers (`q_shared.h` or equivalent)
- `qboolean` — defined in `q_shared.h`
- `rand()` — standard C library (used in `random()` macro)
- `BOTLIB` — compile-time preprocessor flag controlling `FindQuakeFile` signature
- `MAX_PATH` — guarded; fallback defined here as 64 if not previously defined
