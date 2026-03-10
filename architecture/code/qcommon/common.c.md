# code/qcommon/common.c

## File Purpose
The central nervous system of Quake III Arena's engine, providing initialization, shutdown, per-frame orchestration, memory management (zone and hunk allocators), error handling, event loop, and shared utilities used by both client and server subsystems.

## Core Responsibilities
- Engine startup (`Com_Init`) and shutdown (`Com_Shutdown`) sequencing
- Per-frame loop (`Com_Frame`): event dispatch, server tick, client tick, timing
- Zone memory allocator (two pools: `mainzone`, `smallzone`) with tag-based freeing
- Hunk memory allocator (dual-ended stack: low/high with temp and permanent regions)
- Error handling (`Com_Error`) with `longjmp`-based recovery for non-fatal drops
- Event system: push/pop queue with optional journal file recording/replay
- Command-line parsing and startup variable injection
- Console tab-completion infrastructure

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `memblock_t` | struct | Node in zone allocator doubly-linked list; tracks size, tag, ZONEID |
| `memzone_t` | struct | Zone pool header; contains block list, rover pointer, size/used counters |
| `memstatic_t` | struct | Pre-allocated static blocks for empty string and digit strings |
| `hunkHeader_t` | struct | Header prepended to each temp hunk allocation; magic + size |
| `hunkUsed_t` | struct | Tracks mark/permanent/temp/tempHighwater offsets for one hunk end |
| `hunkblock_t` | struct | Debug-only linked list node for hunk allocation tracking |
| `zonedebug_t` | struct | Debug-only label/file/line annotation on zone blocks |
| `e_prefetch` | enum | Prefetch hint type for x86 `Com_Prefetch` (PRE_READ / PRE_WRITE / PRE_READ_WRITE) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mainzone` | `memzone_t *` | global | Primary dynamic allocation pool |
| `smallzone` | `memzone_t *` | global | Small-allocation pool (512 KB) to avoid fragmenting mainzone |
| `s_hunkData` | `byte *` | static | Raw hunk buffer pointer |
| `s_hunkTotal` | `int` | static | Total hunk bytes allocated |
| `hunk_low`, `hunk_high` | `hunkUsed_t` | static | Low/high watermark state for dual-ended hunk |
| `hunk_permanent`, `hunk_temp` | `hunkUsed_t *` | static | Pointers to the current permanent/temp hunk ends |
| `abortframe` | `jmp_buf` | global | longjmp target for ERR_DROP recovery |
| `com_errorEntered` | `qboolean` | global | Recursion guard for `Com_Error` |
| `com_fullyInitialized` | `qboolean` | global | Guards config writes before init completes |
| `com_pushedEvents` | `sysEvent_t[1024]` | static | Ring buffer for pre-queued events |
| `com_pushedEventsHead/Tail` | `int` | static | Ring buffer head/tail indices |
| `rd_buffer`, `rd_flush` | `char*`, fn ptr | static | Output redirection target (e.g., for RCON) |
| `logfile` | `fileHandle_t` | static | qconsole.log file handle |
| `com_journalFile`, `com_journalDataFile` | `fileHandle_t` | global | Journal record/replay file handles |
| `cl_cdkey` | `char[34]` | global | CD key storage (dedicated vs. client default differs) |
| `emptystring`, `numberstring[]` | `memstatic_t` | global | Static zone blocks for single-char/empty strings |

## Key Functions / Methods

### Com_Init
- **Signature:** `void Com_Init( char *commandLine )`
- **Purpose:** Full engine bootstrap sequence.
- **Inputs:** Raw command line string (argv[0] excluded).
- **Outputs/Return:** void; sets `com_fullyInitialized = qtrue` on success.
- **Side effects:** Allocates all memory pools, registers all cvars and commands, initializes FS, network, VM, SV, CL subsystems.
- **Calls:** `Com_InitSmallZoneMemory`, `Cvar_Init`, `Com_ParseCommandLine`, `Cbuf_Init`, `Com_InitZoneMemory`, `Cmd_Init`, `FS_InitFilesystem`, `Com_InitJournaling`, `Com_InitHunkMemory`, `Sys_Init`, `Netchan_Init`, `VM_Init`, `SV_Init`, `CL_Init`, `CL_StartHunkUsers`.
- **Notes:** Uses `setjmp(abortframe)` to catch ERR_DROP during init; fatal on error.

### Com_Frame
- **Signature:** `void Com_Frame( void )`
- **Purpose:** Main per-frame loop tick: events → server → client, with FPS cap and timing.
- **Inputs:** None (reads global cvars).
- **Outputs/Return:** void.
- **Side effects:** Updates `com_frameTime`, `com_frameNumber`, `time_game`, `time_frontend`, `time_backend`; may init/shutdown CL if `dedicated` changes.
- **Calls:** `Com_WriteConfiguration`, `Com_EventLoop`, `Cbuf_Execute`, `Com_ModifyMsec`, `SV_Frame`, `CL_Frame`.
- **Notes:** Uses `setjmp(abortframe)` to silently absorb ERR_DROP within a frame.

### Com_Error
- **Signature:** `void QDECL Com_Error( int code, const char *fmt, ... )`
- **Purpose:** Centralized error dispatch; recoverable drops use `longjmp`, fatals call `Sys_Error`.
- **Inputs:** `code` (ERR_FATAL / ERR_DROP / ERR_DISCONNECT / etc.), format string.
- **Side effects:** May call `SV_Shutdown`, `CL_Disconnect`, `CL_FlushMemory`, `Com_Shutdown`, `Sys_Error`. Sets `com_errorMessage`.
- **Notes:** Escalates rapid ERR_DROP bursts (>3 in 100 ms) to ERR_FATAL.

### Z_TagMalloc / Z_Malloc / S_Malloc
- **Signature:** `void *Z_TagMalloc( int size, int tag )` / `void *Z_Malloc( int size )` / `void *S_Malloc( int size )`
- **Purpose:** Zone allocator; first-fit rover scan, optional fragmentation split.
- **Side effects:** Modifies `zone->rover`, `zone->used`; writes ZONEID trash-test sentinel.
- **Notes:** `Z_Malloc` zero-fills; `S_Malloc` uses `smallzone`; `Z_TagMalloc` does not zero-fill.

### Z_Free
- **Signature:** `void Z_Free( void *ptr )`
- **Purpose:** Release a zone block; merges adjacent free blocks; skips TAG_STATIC.
- **Side effects:** Zeroes freed payload with `0xaa`, merges prev/next free blocks, updates `zone->used`.

### Hunk_Alloc
- **Signature:** `void *Hunk_Alloc( int size, ha_pref preference )`
- **Purpose:** Permanent hunk allocation from low or high end, cache-line aligned.
- **Side effects:** Advances `hunk_permanent->permanent`; may swap banks via `Hunk_SwapBanks`.
- **Notes:** ERR_DROP on OOM. Zero-fills result.

### Hunk_AllocateTempMemory / Hunk_FreeTempMemory
- **Purpose:** Stack-discipline temporary allocations from hunk_temp end. Falls back to `Z_Malloc` if hunk not yet initialized.
- **Notes:** Free must be LIFO; non-LIFO frees log a warning but do not recover memory until `Hunk_ClearTempMemory`.

### Com_EventLoop
- **Signature:** `int Com_EventLoop( void )`
- **Purpose:** Drain event queue; dispatch key/mouse/console/packet events to CL or SV. Returns last event time.
- **Calls:** `Com_GetEvent`, `CL_KeyEvent`, `CL_MouseEvent`, `SV_PacketEvent`, `CL_PacketEvent`, `NET_GetLoopPacket`.

### Com_Printf / Com_DPrintf
- **Purpose:** Engine-wide print; routes to console, `Sys_Print`, and optionally `qconsole.log`. `Com_DPrintf` gates on `com_developer`.
- **Side effects:** May open `qconsole.log` lazily; supports output redirection via `rd_buffer`.

### Field_CompleteCommand
- **Signature:** `void Field_CompleteCommand( field_t *field )`
- **Purpose:** Tab-complete a console input field against registered commands and cvars.
- **Side effects:** Modifies `field->buffer` and `field->cursor`; prints matches via `Com_Printf`.

## Control Flow Notes
- **Init:** `main`/`WinMain` → `Com_Init` (one-shot sequential bootstrap).
- **Frame:** Platform main loop calls `Com_Frame` once per iteration. `setjmp` at the top of `Com_Frame` catches ERR_DROP and returns cleanly.
- **Shutdown:** `Com_Quit_f` → `SV_Shutdown` → `CL_Shutdown` → `Com_Shutdown` → `Sys_Quit`.
- The hunk is cleared by `Hunk_Clear` between map loads; zone persists across maps.

## External Dependencies
- `../game/q_shared.h`, `qcommon.h` — shared types, cvar, filesystem, net, VM interfaces
- `<setjmp.h>` — `setjmp`/`longjmp` for ERR_DROP recovery
- `<netinet/in.h>` (Linux/macOS) or `<winsock.h>` (Win32) — network byte order
- **Defined elsewhere (called here):** `CL_*`, `SV_*`, `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_*`, `NET_*`, `VM_*`, `Sys_*`, `Netchan_Init`, `MSG_*`, `Key_WriteBindings`, `UI_usesUniqueCDKey`, `CIN_CloseAllVideos`
