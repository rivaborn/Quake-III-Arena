# code/qcommon/unzip.h

## File Purpose
Public header for a ZIP file reading library (unzip), adapted from the zlib/minizip project for use in Quake III Arena's virtual filesystem. It declares all types, internal structures, error codes, and the full API for opening, navigating, and decompressing entries within ZIP-format `.pk3` files.

## Core Responsibilities
- Define the opaque `unzFile` handle type (with optional strict-typing via `STRICTUNZIP`)
- Declare metadata structures for ZIP global info, per-file info, and date/time
- Expose the internal streaming state (`z_stream`, `file_in_zip_read_info_s`, `unz_s`) directly in the header
- Define error/status codes for all unzip operations
- Declare the full public API for ZIP navigation and decompression

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `unzFile` | typedef (void* or opaque struct pointer) | Handle to an open ZIP archive |
| `tm_unz` | struct | Date/time fields for a ZIP entry (DOS-style) |
| `unz_global_info` | struct | Top-level ZIP metadata: entry count and comment size |
| `unz_file_info` | struct | Per-entry metadata: sizes, CRC, compression method, attributes, date |
| `unz_file_info_internal` | struct | Internal: byte offset of the entry's local header |
| `z_stream` | struct | zlib inflate stream state; buffers, I/O pointers, allocator hooks |
| `file_in_zip_read_info_s` | struct | Internal per-file decompression context: CRC state, stream, buffer, file pointer |
| `unz_s` | struct | Internal ZIP archive state: file handle, central directory position, current file cursor, temp buffer |

## Global / File-Static State
None.

## Key Functions / Methods

### unzOpen / unzReOpen
- Signature: `unzFile unzOpen(const char *path)` / `unzFile unzReOpen(const char *path, unzFile file)`
- Purpose: Open a ZIP archive by filesystem path; `unzReOpen` reuses an existing handle.
- Inputs: Filesystem path string; optionally an existing `unzFile` for re-open.
- Outputs/Return: `unzFile` handle on success, `NULL` on failure.
- Side effects: Allocates `unz_s` and reads central directory from disk.
- Calls: Not inferable from this file.
- Notes: Failure indicates missing file or invalid ZIP format.

### unzClose
- Signature: `int unzClose(unzFile file)`
- Purpose: Close an open ZIP archive and free associated resources.
- Inputs: Open `unzFile` handle.
- Outputs/Return: `UNZ_OK` or error code.
- Side effects: Frees `unz_s`; any open current file must be closed first.

### unzGoToFirstFile / unzGoToNextFile
- Signature: `int unzGoToFirstFile(unzFile file)` / `int unzGoToNextFile(unzFile file)`
- Purpose: Iterate through the central directory; sets the current file cursor.
- Outputs/Return: `UNZ_OK` or `UNZ_END_OF_LIST_OF_FILE`.

### unzLocateFile
- Signature: `int unzLocateFile(unzFile file, const char *szFileName, int iCaseSensitivity)`
- Purpose: Search the central directory for a named entry; makes it current.
- Inputs: Archive handle, filename string, case-sensitivity flag (`UNZ_CASESENSITIVE`, `UNZ_NOTCASESENSITIVE`, `UNZ_OSDEFAULTCASE`).
- Outputs/Return: `UNZ_OK` if found; `UNZ_END_OF_LIST_OF_FILE` if not.

### unzGetCurrentFileInfo
- Signature: `int unzGetCurrentFileInfo(unzFile file, unz_file_info*, char*, unsigned long, void*, unsigned long, char*, unsigned long)`
- Purpose: Retrieve metadata (size, CRC, date, compression method) and optional filename/comment/extra-field for the current entry.
- Outputs/Return: `UNZ_OK` or error code.

### unzOpenCurrentFile / unzCloseCurrentFile
- Signature: `int unzOpenCurrentFile(unzFile file)` / `int unzCloseCurrentFile(unzFile file)`
- Purpose: Open the current entry for decompressed reading; close and verify CRC on completion.
- Side effects: Allocates/frees `file_in_zip_read_info_s`; initializes zlib inflate stream.
- Notes: `unzCloseCurrentFile` returns `UNZ_CRCERROR` if CRC mismatch is detected after full read.

### unzReadCurrentFile
- Signature: `int unzReadCurrentFile(unzFile file, void *buf, unsigned len)`
- Purpose: Decompress and read bytes from the currently open entry.
- Inputs: Destination buffer and length.
- Outputs/Return: Number of bytes copied; `0` at EOF; negative error code on failure.

### unzStringFileNameCompare
- Signature: `int unzStringFileNameCompare(const char *fileName1, const char *fileName2, int iCaseSensitivity)`
- Purpose: Platform-aware filename comparison used during locate operations.

**Notes:** `unztell`, `unzeof`, `unzGetLocalExtrafield`, `unzGetGlobalInfo`, `unzGetGlobalComment`, `unzGetCurrentFileInfoPosition`, and `unzSetCurrentFileInfoPosition` are trivial position/metadata accessors.

## Control Flow Notes
This header is consumed by `code/qcommon/files.c` (the Q3 virtual filesystem layer). At engine init, `.pk3` files are opened via `unzOpen`; their central directories are enumerated with `unzGoToFirstFile`/`unzGoToNextFile` to build an in-memory file index. At runtime, individual assets are located with `unzLocateFile`, opened with `unzOpenCurrentFile`, read with `unzReadCurrentFile`, and closed with `unzCloseCurrentFile`.

## External Dependencies
- `<stdio.h>` — `FILE*` used directly in `unz_s` and `file_in_zip_read_info_s`
- `struct internal_state` — forward-declared; defined in zlib internals (`zconf.h`/`zlib.h`), not in this file
- `Z_ERRNO` — zlib error code macro, defined externally (zlib.h); aliased as `UNZ_ERRNO`
- Implementation defined in `code/qcommon/unzip.c`
