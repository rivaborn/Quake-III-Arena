# code/bspc/l_qfiles.h

## File Purpose
Header defining the interface for locating, identifying, and loading Quake engine asset files across multiple container formats (PAK, PK3/ZIP, raw filesystem). It provides a unified abstraction over Quake 1/2/3/Sin pack file types for use by the BSPC BSP compiler tool.

## Core Responsibilities
- Define bitmask constants identifying Quake file types (BSP, MD2, MD3, AAS, WAL, etc.)
- Define canonical uppercase file extension strings for each type
- Declare the `quakefile_t` linked-list node representing a located asset file
- Declare the `dsinpackfile_t` structure for Sin pack directory entries
- Expose API for file-type detection by extension or filename
- Expose API for glob-style file searching across pack and filesystem sources
- Expose API for loading or partially reading a located Quake file into memory

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `dsinpackfile_t` | struct | Directory entry for a Sin (`.SIN`) pack file; holds name, offset, and length |
| `quakefile_t` | struct | Unified descriptor for any locatable Quake asset — raw file or entry inside a PAK/PK3; forms a singly-linked list |
| `unz_s` | struct (from unzip.h) | Internal zlib/minizip state embedded in `quakefile_t.zipinfo` for PK3 entries |

## Global / File-Static State
None.

## Key Functions / Methods

### QuakeFileTypeExtension
- Signature: `char *QuakeFileTypeExtension(int type)`
- Purpose: Maps a `QFILETYPE_*` bitmask constant to its canonical extension string (e.g., `QFILETYPE_BSP` → `".BSP"`).
- Inputs: `type` — one of the `QFILETYPE_*` constants.
- Outputs/Return: Pointer to a static extension string; `QFILEEXT_UNKNOWN` (`""`) if unrecognized.
- Side effects: None inferable.
- Calls: Not inferable from this file.
- Notes: Return value should be treated as read-only.

### QuakeFileExtensionType
- Signature: `int QuakeFileExtensionType(char *extension)`
- Purpose: Reverse of `QuakeFileTypeExtension`; maps an extension string to its `QFILETYPE_*` constant.
- Inputs: `extension` — a file extension string (e.g., `".BSP"`).
- Outputs/Return: Matching `QFILETYPE_*` constant, or `QFILETYPE_UNKNOWN`.
- Side effects: None inferable.
- Calls: Not inferable from this file.

### QuakeFileType
- Signature: `int QuakeFileType(char *filename)`
- Purpose: Derives file type by extracting and matching the extension of a full filename.
- Inputs: `filename` — full or partial file path.
- Outputs/Return: `QFILETYPE_*` constant.
- Side effects: None inferable.
- Calls: Likely calls `QuakeFileExtensionType` internally.

### FileFilter
- Signature: `int FileFilter(char *filter, char *filename, int casesensitive)`
- Purpose: Tests whether a filename matches a glob/wildcard filter pattern.
- Inputs: `filter` — pattern string; `filename` — candidate name; `casesensitive` — boolean flag.
- Outputs/Return: Non-zero if filename matches filter.
- Side effects: None inferable.
- Calls: Not inferable from this file.

### FindQuakeFiles
- Signature: `quakefile_t *FindQuakeFiles(char *filter)`
- Purpose: Searches the filesystem and/or PAK/PK3 archives for all files matching `filter`, returning a linked list of `quakefile_t` descriptors.
- Inputs: `filter` — glob pattern potentially including pack-relative paths.
- Outputs/Return: Head of a `quakefile_t *` linked list; `NULL` if none found.
- Side effects: Allocates heap memory for the list nodes.
- Calls: `FileFilter`, `QuakeFileType`; likely opens PAK/PK3 using `unzOpen` family.

### LoadQuakeFile
- Signature: `int LoadQuakeFile(quakefile_t *qf, void **bufferptr)`
- Purpose: Fully loads the file described by `qf` into a newly allocated buffer.
- Inputs: `qf` — a `quakefile_t` descriptor; `bufferptr` — out-parameter receives the allocated buffer pointer.
- Outputs/Return: File length in bytes on success; ≤0 on failure.
- Side effects: Allocates heap memory; may open/close file handles or ZIP entries.
- Calls: Likely calls `ReadQuakeFile` or zlib unzip functions (`unzOpenCurrentFile`, `unzReadCurrentFile`).

### ReadQuakeFile
- Signature: `int ReadQuakeFile(quakefile_t *qf, void *buffer, int offset, int length)`
- Purpose: Reads a byte range `[offset, offset+length)` from the file described by `qf` into a caller-supplied buffer (partial/random-access read).
- Inputs: `qf` — file descriptor; `buffer` — destination; `offset` — byte offset within the file; `length` — byte count.
- Outputs/Return: Number of bytes read; ≤0 on error.
- Side effects: I/O; may seek within a PAK file or decompress from a PK3 stream.
- Calls: Standard `fseek`/`fread` or zlib unzip API depending on `qf->zipfile`.

## Control Flow Notes
This is a pure header — no control flow is defined here. The functions declared are used by the BSPC tool during its asset-loading phase (before BSP compilation begins) to locate and read map, model, and AAS source files from Quake game directories. `FindQuakeFiles` is the entry point; callers iterate the resulting list and invoke `LoadQuakeFile` or `ReadQuakeFile` per entry.

## External Dependencies
- `../qcommon/unzip.h` — provides `unz_s` (embedded by value in `quakefile_t.zipinfo`) and the full minizip API used by the implementation (`l_qfiles.c`).
- `_MAX_PATH` — conditionally defined here as 1024 if not already provided by the platform.
- Implementation (`l_qfiles.c`) defined elsewhere; all function bodies external to this file.
