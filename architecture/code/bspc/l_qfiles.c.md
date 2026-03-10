# code/bspc/l_qfiles.c

## File Purpose
Provides utilities for discovering and loading Quake engine game files across multiple archive formats (PAK, PK3/ZIP, loose files) on both Windows and POSIX platforms. It abstracts the differences between id-format PAK files, Sin PAK files, and ZIP-based PK3 packages into a unified `quakefile_t` linked-list interface.

## Core Responsibilities
- Map file extensions to Quake file type enum constants (bidirectional lookup)
- Implement glob/wildcard pattern matching (`FileFilter`, `StringContains`) for archive entry filtering
- Enumerate files inside ZIP/PK3 archives matching a filter
- Enumerate files inside id PAK and Sin PAK archives matching a filter
- Recursively search filesystem paths (Win32 `FindFirstFile`/POSIX `glob`) to locate loose files or pak containers
- Parse a composite filter string (`path/to/archive.pak/subpath/*.bsp`) and dispatch to the correct loader
- Load or partially read a matched `quakefile_t` into memory, handling both ZIP streams and raw file offsets

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `qfile_exttyp_t` | struct (`typedef`) | Pairs a file extension string with its integer type constant |
| `quakefile_t` | struct (defined in `l_qfiles.h`) | Describes a located game file: archive path, internal name, byte offset, length, type, zip metadata, linked-list pointer |
| `dpackheader_t` | struct (defined elsewhere) | On-disk header for id/Sin PAK archives |
| `dpackfile_t` | struct (defined elsewhere) | id PAK directory entry (name, offset, length) |
| `dsinpackfile_t` | struct (defined elsewhere) | Sin PAK directory entry (wider name field) |
| `unz_s` / `unzFile` | struct / typedef (minizip) | Minizip handle; cast directly to copy zip seek state into `quakefile_t::zipinfo` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `quakefiletypes[]` | `qfile_exttyp_t[]` | file-static (translation-unit global) | Null-terminated table mapping extension strings to `QFILETYPE_*` constants |

## Key Functions / Methods

### QuakeFileExtensionType
- **Signature:** `int QuakeFileExtensionType(char *extension)`
- **Purpose:** Linear search of `quakefiletypes[]` to return the integer type for a given extension string.
- **Inputs:** `extension` — string such as `".bsp"`
- **Outputs/Return:** Matching `QFILETYPE_*` constant, or `QFILETYPE_UNKNOWN`
- **Side effects:** None
- **Calls:** `stricmp`

### QuakeFileType
- **Signature:** `int QuakeFileType(char *filename)`
- **Purpose:** Extracts the extension from a full filename and delegates to `QuakeFileExtensionType`.
- **Inputs:** Full or partial file path
- **Outputs/Return:** `QFILETYPE_*` constant
- **Calls:** `ExtractFileExtension`, `QuakeFileExtensionType`

### FileFilter
- **Signature:** `int FileFilter(char *filter, char *filename, int casesensitive)`
- **Purpose:** Glob-style pattern match: supports `*`, `?`, and `[…]` character classes.
- **Inputs:** `filter` — pattern; `filename` — candidate string; `casesensitive` — boolean
- **Outputs/Return:** Non-zero if filename matches, 0 otherwise
- **Side effects:** None
- **Calls:** `StringContains`, `toupper`, `strlen`
- **Notes:** `[[` is treated as a literal `[`. Does not anchor the end of the string after a trailing `*`.

### FindQuakeFilesInZip
- **Signature:** `quakefile_t *FindQuakeFilesInZip(char *zipfile, char *filter)`
- **Purpose:** Opens a PK3/ZIP and iterates every entry, collecting those matching `filter` into a heap-allocated linked list of `quakefile_t`.
- **Inputs:** Path to zip, glob filter
- **Outputs/Return:** Head of `quakefile_t` linked list, or NULL
- **Side effects:** Heap allocation (`malloc`); calls `Error` on OOM
- **Calls:** `unzOpen`, `unzGetGlobalInfo`, `unzGoToFirstFile`, `unzGetCurrentFileInfo`, `FileFilter`, `ConvertPath`, `malloc`, `memset`, `memcpy`, `QuakeFileType`, `unzGoToNextFile`, `unzClose`
- **Notes:** Copies `unz_s` struct by value into `qf->zipinfo` to preserve seek position — relies on minizip internals.

### FindQuakeFilesInPak
- **Signature:** `quakefile_t *FindQuakeFilesInPak(char *pakfile, char *filter)`
- **Purpose:** Reads a PAK directory (id or Sin format), filters entries, and returns a linked list of matching `quakefile_t`.
- **Inputs:** Path to .pak or .sin file, glob filter
- **Outputs/Return:** Head of `quakefile_t` linked list, or NULL
- **Side effects:** File I/O; heap allocation; `Warning` on invalid file
- **Calls:** `fopen`, `fread`, `fseek`, `fclose`, `malloc`, `free`, `LittleLong`, `ConvertPath`, `FileFilter`, `QuakeFileType`, `Warning`, `Error`
- **Notes:** Normalizes id PAK entries into `dsinpackfile_t` layout before filtering.

### FindQuakeFilesWithPakFilter
- **Signature:** `quakefile_t *FindQuakeFilesWithPakFilter(char *pakfilter, char *filter)`
- **Purpose:** Filesystem glob for pak/pk3/directory matches, then dispatches to `FindQuakeFilesInZip`, `FindQuakeFilesInPak`, or recursive directory scan. If `pakfilter` is NULL, enumerates loose files matching `filter`.
- **Inputs:** `pakfilter` — optional glob for archive files; `filter` — internal or filesystem file filter
- **Outputs/Return:** Merged `quakefile_t` linked list
- **Side effects:** Platform I/O (Win32 `FindFirstFile`/POSIX `glob`); heap allocation
- **Calls:** `FindQuakeFilesInZip`, `FindQuakeFilesInPak`, `StringContains`, `AppendPathSeperator`, `stat`/`_stat`, `glob`/`FindFirstFile`

### FindQuakeFiles
- **Signature:** `quakefile_t *FindQuakeFiles(char *filter)`
- **Purpose:** Top-level entry point; parses a composite filter string to extract an optional pak sub-path and delegates.
- **Inputs:** Filter string (may embed `.pak` or `.pk3` followed by an internal path)
- **Outputs/Return:** `quakefile_t` linked list
- **Calls:** `ConvertPath`, `StringContains`, `FindQuakeFilesWithPakFilter`

### LoadQuakeFile
- **Signature:** `int LoadQuakeFile(quakefile_t *qf, void **bufferptr)`
- **Purpose:** Fully reads a `quakefile_t` into a newly allocated buffer, handling both ZIP and flat-file sources.
- **Inputs:** Populated `quakefile_t`, pointer to receive buffer
- **Outputs/Return:** Byte count read; `*bufferptr` set to allocated memory
- **Side effects:** Heap allocation (`GetMemory`); file/zip I/O
- **Calls:** `unzOpen`, `unzOpenCurrentFile`, `unzReadCurrentFile`, `unzCloseCurrentFile`, `unzClose`, `SafeOpenRead`, `fseek`, `Q_filelength`, `GetMemory`, `SafeRead`, `fclose`

### ReadQuakeFile
- **Signature:** `int ReadQuakeFile(quakefile_t *qf, void *buffer, int offset, int length)`
- **Purpose:** Partial read from a `quakefile_t` at a given byte offset into a caller-supplied buffer.
- **Inputs:** `quakefile_t`, caller buffer, intra-file offset, byte count
- **Outputs/Return:** Bytes read
- **Side effects:** File/zip I/O; for ZIP files, skips `offset` bytes by consuming them via 1024-byte chunks
- **Calls:** `unzOpen`, `unzOpenCurrentFile`, `unzReadCurrentFile`, `unzCloseCurrentFile`, `unzClose`, `SafeOpenRead`, `fseek`, `SafeRead`, `fclose`

## Control Flow Notes
This file is a utility/support module with no frame or tick callback. It is invoked during the **initialization / map-loading** phase of the BSPC tool when the tool needs to locate and load BSP, MAP, AAS, or model files from game data directories or pack archives. `FindQuakeFiles` → `FindQuakeFilesWithPakFilter` → `FindQuakeFilesInPak`/`FindQuakeFilesInZip` forms the discovery pipeline; `LoadQuakeFile`/`ReadQuakeFile` are called subsequently by map/model loaders.

## External Dependencies
- `qbsp.h` (transitively pulls in all BSPC headers)
- `l_qfiles.h` — declares `quakefile_t`, `QFILETYPE_*`, `QFILEEXT_*` constants
- `unzip.h` / minizip — `unzFile`, `unz_s`, `unzOpen`, `unzGetGlobalInfo`, etc.
- `q2files.h` — `dpackheader_t`, `dpackfile_t`, `dsinpackfile_t`, `IDPAKHEADER`, `SINPAKHEADER`
- `l_cmd.h` / `l_utils.h` — `ExtractFileExtension`, `ConvertPath`, `AppendPathSeperator`, `SafeOpenRead`, `SafeRead`, `Q_filelength`, `GetMemory`, `Error`, `Warning`
- Win32: `<windows.h>`, `FindFirstFile`/`FindNextFile`, `_splitpath`, `_stat`
- POSIX: `<glob.h>`, `<unistd.h>`, `glob`/`globfree`, `stat`
- `LittleLong` — byte-order conversion, defined elsewhere
