# code/qcommon/files.c

## File Purpose
Implements Quake III Arena's handle-based virtual filesystem, which transparently merges content from multiple source directories and `.pk3` (zip) archives. It manages all file I/O for the engine, enforcing path security, pure-server validation, and demo/restricted-mode restrictions.

## Core Responsibilities
- Initialize and shut down the search path hierarchy (base/cd/home paths, mod directories)
- Load and index `.pk3` zip archives into hash-table-backed `pack_t` structures
- Resolve file reads by walking `fs_searchpaths` in priority order (pk3 before dir, newer pak before older)
- Enforce pure-server mode (only allow files from server-approved pak checksums)
- Track pak reference flags (`FS_GENERAL_REF`, `FS_CGAME_REF`, `FS_UI_REF`, `FS_QAGAME_REF`) for sv_pure negotiation
- Provide directory listing, mod enumeration, and file copy/rename operations
- Support journal-based replay of config file reads

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fileInPack_t` | struct | Single file entry inside a pk3: name pointer, zip position, hash chain next |
| `pack_t` | struct | Loaded pk3 archive: filename, handle, checksums, file count, hash table of `fileInPack_t` |
| `directory_t` | struct | Plain directory search entry: OS path + game subdirectory |
| `searchpath_t` | struct | Linked-list node holding either a `pack_t*` or `directory_t*` |
| `fileHandleData_t` | struct | Per-handle state: underlying FILE/unzFile union, zip flag, size, stream flag, name |
| `qfile_gut` / `qfile_ut` | union/struct | Wraps FILE* or unzFile in a unified handle slot |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `fs_searchpaths` | `searchpath_t*` | static | Head of the search path linked list |
| `fsh[MAX_FILE_HANDLES]` | `fileHandleData_t[]` | static | Array of open file handle slots |
| `fs_gamedir` | `char[]` | static | Current active game directory name |
| `fs_debug/homepath/basepath/cdpath/…` | `cvar_t*` | static | Filesystem configuration cvars |
| `fs_numServerPaks` / `fs_serverPaks[]` | `int` / `int[]` | static | Pure-server approved pak checksums |
| `fs_numServerReferencedPaks` / arrays | `int` / arrays | static | Server-referenced paks for autodownload |
| `fs_checksumFeed` | `int` | static | Feed value mixed into pure checksums |
| `fs_fakeChkSum` | `int` | static | Injected into pure checksum when non-pure files are read |
| `fs_reordered` | `qboolean` | static | Whether search path was reordered for pure server |
| `lastValidBase` / `lastValidGame` | `char[]` | global | Fallback paths on restart failure |

## Key Functions / Methods

### FS_InitFilesystem
- **Signature:** `void FS_InitFilesystem(void)`
- **Purpose:** One-time startup; reads command-line overrides for path cvars, calls `FS_Startup(BASEGAME)`, sets restrictions, validates `default.cfg` existence.
- **Inputs:** None (reads command-line via `Com_StartupVariable`)
- **Outputs/Return:** None
- **Side effects:** Populates `fs_searchpaths`; sets `lastValidBase`/`lastValidGame`
- **Calls:** `Com_StartupVariable`, `FS_Startup`, `FS_SetRestrictions`, `FS_ReadFile`

### FS_Startup
- **Signature:** `static void FS_Startup(const char *gameName)`
- **Purpose:** Registers filesystem cvars, calls `FS_AddGameDirectory` for all path/game combos in reverse priority, reads CD key, registers console commands, reorders pure paks.
- **Inputs:** `gameName` — base game or mod directory name
- **Side effects:** Builds `fs_searchpaths`; registers `path`, `dir`, `fdir`, `touchFile` commands

### FS_AddGameDirectory
- **Signature:** `static void FS_AddGameDirectory(const char *path, const char *dir)`
- **Purpose:** Adds a directory entry and all its `.pk3` files (sorted, so pak1 > pak0) to the front of `fs_searchpaths`.
- **Inputs:** OS base path, game subdirectory
- **Side effects:** Allocates `searchpath_t`, `directory_t`, `pack_t` via `Z_Malloc`; calls `FS_LoadZipFile`

### FS_LoadZipFile
- **Signature:** `static pack_t *FS_LoadZipFile(char *zipfile, const char *basename)`
- **Purpose:** Opens a pk3, enumerates all entries, builds hash table, computes regular and pure checksums.
- **Inputs:** OS path to zip, basename string
- **Outputs/Return:** Newly allocated `pack_t*`, or NULL on error
- **Side effects:** `Z_Malloc` for `pack_t`, `buildBuffer`, `fs_headerLongs`; increments `fs_packFiles`
- **Calls:** `unzOpen`, `unzGetGlobalInfo`, `unzGetCurrentFileInfo`, `Com_BlockChecksum`, `Com_BlockChecksumKey`

### FS_FOpenFileRead
- **Signature:** `int FS_FOpenFileRead(const char *filename, fileHandle_t *file, qboolean uniqueFILE)`
- **Purpose:** Primary read-open function; walks `fs_searchpaths`, checks pure list, marks pak references, opens zip entry or plain file.
- **Inputs:** qpath filename, output handle pointer, uniqueFILE flag
- **Outputs/Return:** File size on success; -1 on failure; `qtrue`/`qfalse` if `file==NULL` (existence check)
- **Side effects:** Sets `fsh[*file]`; sets `pak->referenced` flags; may copy file from cd path
- **Calls:** `FS_HashFileName`, `FS_FilenameCompare`, `FS_PakIsPure`, `unzReOpen`, `unzOpenCurrentFile`, `FS_ShiftedStrStr`

### FS_ReadFile
- **Signature:** `int FS_ReadFile(const char *qpath, void **buffer)`
- **Purpose:** Convenience function that reads an entire file into a `Hunk_AllocateTempMemory` buffer (null-terminated). Handles journal playback for `.cfg` files.
- **Inputs:** qpath, pointer-to-buffer (NULL for size-only query)
- **Outputs/Return:** File length; -1 if not found
- **Side effects:** Increments `fs_loadCount`, `fs_loadStack`; writes to journal if active
- **Calls:** `FS_FOpenFileRead`, `FS_Read`, `Hunk_AllocateTempMemory`, `FS_FCloseFile`

### FS_Shutdown
- **Signature:** `void FS_Shutdown(qboolean closemfp)`
- **Purpose:** Closes all open handles, frees all search path nodes, pack buffers, and zip handles; nulls `fs_searchpaths`.
- **Side effects:** All memory freed; console commands removed

### FS_Restart
- **Signature:** `void FS_Restart(int checksumFeed)`
- **Purpose:** Shutdown + re-startup with a new checksum feed (called when connecting to a pure server or changing game).
- **Side effects:** Full teardown and rebuild of `fs_searchpaths`; may exec `q3config.cfg`

### FS_ReferencedPakPureChecksums
- **Signature:** `const char *FS_ReferencedPakPureChecksums(void)`
- **Purpose:** Builds the ordered pure-checksum string sent by clients to the server for sv_pure validation (`cgame ui @ general...`).
- **Outputs/Return:** Static string (not re-entrant)
- **Notes:** Incorporates `fs_fakeChkSum` to poison the result if non-pure files were accessed.

### FS_PureServerSetLoadedPaks
- **Signature:** `void FS_PureServerSetLoadedPaks(const char *pakSums, const char *pakNames)`
- **Purpose:** Receives the server's list of approved pak checksums; stores in `fs_serverPaks[]`; triggers `FS_Restart` if a reorder is needed.

## Control Flow Notes
- **Init:** `FS_InitFilesystem` called once from `Com_Init` before any other subsystem.
- **Per-frame:** No per-frame presence; file I/O is on-demand.
- **Restart:** `FS_ConditionalRestart` / `FS_Restart` called by the client when receiving a new gamestate or when `fs_game` changes.
- **Shutdown:** `FS_Shutdown` called from `Com_Shutdown` and before `FS_Restart`.

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `fileHandle_t`, `cvar_t`, `fsMode_t`, `Q_str*` utilities
- `qcommon.h` — `Com_Error`, `Com_Printf`, `Cvar_Get`, `Cmd_*`, `Hunk_*`, `Z_Malloc/Free`, `S_ClearSoundBuffer`, `Com_BlockChecksum`
- `unzip.h` — `unzFile`, `unz_s`, `unzOpen/Close/Read/…` (zlib-based zip reading)
- **Defined elsewhere:** `Sys_ListFiles`, `Sys_FreeFileList`, `Sys_Mkdir`, `Sys_DefaultCDPath/InstallPath/HomePath`, `Sys_BeginStreamedFile/EndStreamedFile/StreamedRead/StreamSeek`, `Com_AppendCDKey`, `Com_ReadCDKey`, `Com_FilterPath`
