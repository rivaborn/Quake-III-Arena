# code/qcommon/files.c — Enhanced Analysis

## Architectural Role

`files.c` is the single choke point through which every engine subsystem accesses game data. It sits at the center of `qcommon` and exposes an OS-agnostic, handle-based I/O API that transparently unifies plain directories and `.pk3` ZIP archives under a uniform search-path hierarchy. The renderer imports it through `refimport_t ri.FS_*`; the collision model (`CM_`) reads BSP data through `FS_ReadFile`; botlib receives file access through its `botlib_import_t` vtable; all three VM types (game, cgame, ui) reach it via indexed `trap_*` syscalls that the engine dispatches through `SV_GameSystemCalls`/`CL_CgameSystemCalls`. The pure-server subsystem makes this file directly security-critical: it controls which pak checksums are accepted for multiplayer anti-cheat.

---

## Key Cross-References

### Incoming (who depends on this file)

| Caller | How |
|---|---|
| `code/renderer/tr_init.c`, `tr_image.c`, `tr_shader.c`, `tr_model.c` | `ri.FS_ReadFile`, `ri.FS_FOpenFileRead`, `ri.FS_FreeFile` via `refimport_t` vtable |
| `code/qcommon/cm_load.c` | `FS_ReadFile` for BSP load; the code comment about single-file caching exists precisely because CM and TR both load the same BSP |
| `code/client/cl_main.c`, `cl_cgame.c` | `FS_ConditionalRestart`, `FS_Restart` when receiving gamestate or game dir change |
| `code/server/sv_init.c`, `sv_client.c` | `FS_PureServerSetLoadedPaks`, `FS_ReferencedPakPureChecksums` for sv_pure negotiation |
| `code/server/sv_game.c` | Dispatches `trap_FS_*` syscalls from game VM to `FS_FOpenFileRead`, `FS_Read`, `FS_Write`, `FS_FCloseFile`, `FS_GetFileList` |
| `code/client/cl_cgame.c` | Same for cgame's `trap_FS_*` range |
| `code/client/cl_ui.c` | Same for UI VM |
| `code/botlib/be_interface.c` | File I/O members of `botlib_import_t` (`FS_FOpenFile`, `FS_Read`, `FS_Write`, `FS_FCloseFile`) are filled from this file's functions |
| `code/qcommon/common.c` | `FS_InitFilesystem` at engine boot; `FS_Shutdown` at teardown; `FS_ReadFile` for `q3config.cfg`, `autoexec.cfg` |

### Outgoing (what this file depends on)

- **Platform layer:** `Sys_ListFiles`/`Sys_FreeFileList` for directory enumeration; `Sys_Mkdir` for write path creation; `Sys_DefaultCDPath/InstallPath/HomePath` for initial path resolution; `Sys_BeginStreamedFile`/`StreamedRead`/`StreamSeek`/`EndStreamedFile` for streamed I/O
- **`unzip.h` / `unzip.c`:** Full zlib-backed ZIP API (`unzOpen`, `unzGetGlobalInfo`, `unzGetCurrentFileInfo`, `unzReOpen`, `unzOpenCurrentFile`, `unzReadCurrentFile`, `unzClose`)
- **`qcommon/common.c`:** `Com_Error`, `Com_Printf`, `Com_sprintf`, `Com_BlockChecksum`, `Com_BlockChecksumKey`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Z_Malloc`/`Z_Free`, `Com_AppendCDKey`/`Com_ReadCDKey`
- **`qcommon/cvar.c`:** `Cvar_Get`, `Cvar_Set`
- **`qcommon/cmd.c`:** `Cmd_AddCommand`/`Cmd_RemoveCommand`, `Cmd_Argv`
- **Sound system:** `S_ClearSoundBuffer` called inside `FS_Restart` before teardown — a cross-subsystem coupling that is architecturally notable

---

## Design Patterns & Rationale

**Priority-ordered intrusive linked list (`searchpath_t`):** The list is built by prepending, so last-added = highest priority. `FS_AddGameDirectory` is called in reverse priority order, which means pk3 files inside a higher-priority directory end up at the head. This is a classic "push, then iterate from head" pattern for layered override systems with no need for O(1) lookup by key (iteration is acceptable at load time).

**Hash-table-per-pak for intra-pak lookup:** Each `pack_t` maintains its own `fileInPack_t` hash table sized to a power of two. The hash function deliberately excludes the file extension (stops at `.`) and normalizes slashes — making lookup robust to platform path format variation without requiring normalization at call sites.

**Flip-flop static buffer in `FS_BuildOSPath`:** A `static char ospath[2][MAX_OSPATH]` toggled by `toggle ^= 1` lets two callers hold a returned path simultaneously without collision. This is a deliberate "poor man's caller-owns-the-buffer" that avoids heap allocation in a hot path but is **not thread-safe** and bounds callers to two simultaneous live strings.

**Reference-flag accumulation for pure checksums:** Each `pack_t` carries a `referenced` bitmask (`FS_GENERAL_REF | FS_CGAME_REF | FS_UI_REF | FS_QAGAME_REF`). `FS_FOpenFileRead` sets the appropriate flag each time a file is opened from that pak. `FS_ReferencedPakPureChecksums` later collects these flags to build the ordered `cgame ui @ general` checksum string sent to the server — a clever passive tracking mechanism that requires no explicit registration calls from subsystems.

**Journal-based `.cfg` replay:** `FS_ReadFile` checks `com_journal->integer` and routes `.cfg` reads through an event-log file for deterministic replay. This was used for reproducible bug reporting — a form of primitive record/replay predating modern determinism frameworks.

---

## Data Flow Through This File

```
Boot
  Com_Init
    FS_InitFilesystem
      FS_Startup(gameName)
        Cvar_Get(fs_basepath, fs_cdpath, fs_homepath, ...)
        FS_AddGameDirectory(path, dir)   [called multiple times, reverse priority]
          Sys_ListFiles → sorted pk3 list
          FS_LoadZipFile(pk3) → pack_t with hash table + checksums
          Prepend searchpath_t{pack} to fs_searchpaths
          Prepend searchpath_t{dir}  to fs_searchpaths

Per read (FS_FOpenFileRead)
  qpath
    → FS_HashFileName → bucket in pack->hashTable
    → walk chain comparing FS_FilenameCompare
    → FS_PakIsPure check (against fs_serverPaks[])
    → set pack->referenced flags
    → unzReOpen + unzOpenCurrentFile   (zip path)
       OR fopen                         (dir path)
    → return fileHandle_t + size

Pure server connect
  FS_PureServerSetLoadedPaks(sums, names)
    → store in fs_serverPaks[]
    → if reorder needed: FS_Restart(checksumFeed)
      → FS_Shutdown → FS_Startup
      → S_ClearSoundBuffer (side-effect)

Client disconnect / game change
  FS_ConditionalRestart / FS_Restart
    → same teardown/rebuild cycle
```

The key state transition is `fs_searchpaths = NULL` (shutdown) → populated linked list (startup). During the live state, `fsh[]` tracks open handles and `pack->referenced` accumulates flags. `fs_fakeChkSum` is set non-zero if any read came from a non-pure pak, poisoning the outbound pure-checksum string.

---

## Learning Notes

**Virtual filesystem as anti-piracy and modding infrastructure simultaneously:** The pk3/zip scheme solved packaging, patching (higher-numbered pak wins), restricted demo mode (single-pak allowlist + MD4 checksum), and modding (fs_game directory layer) in one design. Modern engines separate these concerns (asset DBs, DLC manifests, mod APIs) but Q3's approach is impressively compact.

**"Pure server" as a soft anti-cheat primitive:** The checksum comparison in `FS_PakIsPure` is not cryptographically strong (same checksum, different name is accepted per FIXME comment), but it raises the bar enough to deter casual cheating via replacement assets. Modern engines use signed content or server-authoritative streaming instead.

**Single-file caching coupling between CM and renderer:** The comment about `CM_` loading with a "cache" request so the renderer can reuse the same buffer reveals a manual, fragile coordination between two subsystems. Modern engines use a unified asset manager with reference-counted handles.

**`S_ClearSoundBuffer` inside `FS_Restart`:** This is architecturally significant — the filesystem restart triggers sound system cleanup because sound assets may have come from paks that are about to be unmapped. It's an implicit coupling with no abstraction; in a modern engine this would be a subscription/event callback.

**No async I/O:** All reads are synchronous, blocking the main thread. The "streamed" file path (`Sys_BeginStreamedFile`) is the only concession to background I/O, used for cinematic/demo playback. This was acceptable in 1999 given loading screens; modern engines decouple all asset I/O.

**`fileHandle_t` as an index, not a pointer:** `fsh[f]` is a slot table indexed by integer. This prevents dangling-pointer bugs across subsystem boundaries and was idiomatic for engine code of this era (compare: Quake's `cache_user_t`).

---

## Potential Issues

- **`FS_BuildOSPath` is not thread-safe:** The two-slot static buffer assumes single-threaded callers. Any SMP path touching filesystem path construction would race.
- **`fs_serverPaks` is bounded by `MAX_SEARCH_PATHS` (4096) but so is `fs_searchpaths`:** If a server sends more pak checksums than fit, the array silently truncates with no error.
- **`FS_filelength` seeks to end/back:** Called on a non-unique pak FILE returns the `.pk3` container size, not the member file size — documented as a bug risk in the comment but left unfixed.
- **`fs_fakeChkSum` poisoning is bypassable:** A client that patches the checksum accumulation path in memory avoids the poisoning. It is a deterrent, not a guarantee.
