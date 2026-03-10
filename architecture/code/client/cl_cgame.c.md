# code/client/cl_cgame.c

## File Purpose
This file implements the client-side interface layer between the engine and the cgame VM module. It provides the system call dispatch table that the cgame VM invokes to access engine services, and manages cgame VM lifecycle (init, shutdown, per-frame rendering and time updates).

## Core Responsibilities
- Load, initialize, and shut down the cgame VM (`VM_Create`/`VM_Free`)
- Dispatch all cgame system calls (`CL_CgameSystemCalls`) to appropriate engine subsystems
- Expose client state to cgame: snapshots, user commands, game state, GL config, server commands
- Process server commands destined for cgame (`CL_GetServerCommand`) including large config string reassembly (`bcs0/bcs1/bcs2`)
- Manage configstring updates (`CL_ConfigstringModified`) into `cl.gameState`
- Drive server time synchronization and drift correction (`CL_SetCGameTime`, `CL_AdjustTimeDelta`)
- Trigger cgame rendering each frame (`CL_CGameRendering`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `clSnapshot_t` | struct (from client.h) | Internal snapshot with raw parse entity indices; not directly exposed to cgame |
| `snapshot_t` | struct (from cg_public.h) | Cgame-facing snapshot with resolved entity arrays |
| `clientActive_t` (`cl`) | struct/global | All per-connection client state: snapshots, cmds, game state |
| `clientConnection_t` (`clc`) | struct/global | Network/demo connection state, server command buffers |
| `clientStatic_t` (`cls`) | struct/global | Persistent client state: render config, connection state, key catchers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botlib_export` | `botlib_export_t *` | global (extern) | BotLib API used for PC (script parser) calls from cgame |
| `bigConfigString` | `char[BIG_INFO_STRING]` | static (local to `CL_GetServerCommand`) | Assembly buffer for fragmented `bcs0/bcs1/bcs2` config strings |

## Key Functions / Methods

### CL_CgameSystemCalls
- **Signature:** `int CL_CgameSystemCalls( int *args )`
- **Purpose:** Central dispatch for all cgame → engine system calls; registered as callback when VM is created.
- **Inputs:** `args` array where `args[0]` is the trap number (e.g. `CG_PRINT`) and subsequent elements are packed arguments.
- **Outputs/Return:** Integer result (sometimes a `FloatAsInt`-packed float) returned to cgame VM.
- **Side effects:** Delegates to renderer (`re.*`), sound (`S_*`), collision (`CM_*`), filesystem (`FS_*`), cvars, keys, cinematic, and botlib subsystems. May call `Com_Error` on bad trap.
- **Calls:** ~60+ engine functions across all subsystems.
- **Notes:** `VMA(x)` casts arg to pointer via `VM_ArgPtr`; `VMF(x)` reinterprets as float. `CG_R_REGISTERFONT` is missing its `return 0` (bug — falls through to `CG_R_CLEARSCENE`).

### CL_InitCGame
- **Signature:** `void CL_InitCGame( void )`
- **Purpose:** Loads the cgame VM, transitions connection state to `CA_LOADING` then `CA_PRIMED`, calls `CG_INIT`.
- **Inputs:** None (reads `cl.gameState`, `clc`, `cl_connectedToPureServer`, `vm_cgame` cvar).
- **Outputs/Return:** None.
- **Side effects:** Sets `cls.state`; allocates VM; calls `re.EndRegistration()`; may call `Com_TouchMemory()`.
- **Calls:** `VM_Create`, `VM_Call(CG_INIT)`, `re.EndRegistration`, `Sys_LowPhysicalMemory`, `Com_TouchMemory`, `Con_Close`, `Con_ClearNotify`.
- **Notes:** Must only be called from `CL_StartHunkUsers`. Pure-server connections force `VMI_COMPILED`.

### CL_ShutdownCGame
- **Signature:** `void CL_ShutdownCGame( void )`
- **Purpose:** Calls `CG_SHUTDOWN` on the cgame VM and frees it.
- **Side effects:** Clears `KEYCATCH_CGAME`, sets `cls.cgameStarted = qfalse`, nulls `cgvm`.
- **Calls:** `VM_Call(CG_SHUTDOWN)`, `VM_Free`.

### CL_GetServerCommand
- **Signature:** `qboolean CL_GetServerCommand( int serverCommandNumber )`
- **Purpose:** Retrieves and processes a reliable server command by sequence number; handles `disconnect`, `cs`, `bcs0/1/2`, `map_restart`, `clientLevelShot` internally.
- **Side effects:** May call `Com_Error`, `CL_ConfigstringModified`, `Con_ClearNotify`, `Con_Close`, `Cbuf_AddText`. Updates `clc.lastExecutedServerCommand`.
- **Notes:** `bcs0/1/2` are a three-part protocol for oversized config strings; reassembled into `bigConfigString` then rescanned via `goto rescan`.

### CL_ConfigstringModified
- **Signature:** `void CL_ConfigstringModified( void )`
- **Purpose:** Rebuilds `cl.gameState` string table when a `cs` command updates a configstring index.
- **Side effects:** Overwrites `cl.gameState`; calls `CL_SystemInfoChanged` on `CS_SYSTEMINFO` changes.

### CL_SetCGameTime
- **Signature:** `void CL_SetCGameTime( void )`
- **Purpose:** Per-frame update of `cl.serverTime` used by cgame, handling demo playback, pause, time nudge, and first-snapshot detection.
- **Side effects:** Advances `cl.serverTime`; calls `CL_AdjustTimeDelta`, `CL_ReadDemoMessage`, `CL_FirstSnapshot`.
- **Notes:** Clamps `cl_timeNudge` to ±30 ms. For timedemos, forces deterministic 50 ms frame steps.

### CL_AdjustTimeDelta
- **Signature:** `void CL_AdjustTimeDelta( void )`
- **Purpose:** Drifts `cl.serverTimeDelta` toward the true server-to-realtime offset; uses hard reset, fast (halving), or slow (±1–2 ms) modes.
- **Notes:** Only called when new snapshots arrive; no-op during demo playback.

### CL_CGameRendering
- **Signature:** `void CL_CGameRendering( stereoFrame_t stereo )`
- **Purpose:** Calls `CG_DRAW_ACTIVE_FRAME` in the cgame VM to render the current frame.
- **Calls:** `VM_Call(CG_DRAW_ACTIVE_FRAME)`, `VM_Debug(0)`.

### CL_GetSnapshot
- **Signature:** `qboolean CL_GetSnapshot( int snapshotNumber, snapshot_t *snapshot )`
- **Purpose:** Copies a historical snapshot from circular buffers into a cgame-facing `snapshot_t`, resolving parse entity indices.
- **Notes:** Returns `qfalse` if snapshot has fallen out of `PACKET_BACKUP` or entity buffer.

- **Notes (minor helpers):** `CL_GetGameState`, `CL_GetGlconfig`, `CL_GetUserCmd`, `CL_GetCurrentCmdNumber`, `CL_GetParseEntityState`, `CL_GetCurrentSnapshotNumber`, `CL_SetUserCmdValue`, `CL_AddCgameCommand`, `CL_CgameError`, `CL_CM_LoadMap`, `CL_GameCommand`, `CL_FirstSnapshot` — all thin wrappers that copy data or forward calls.

## Control Flow Notes
- **Init:** `CL_StartHunkUsers` → `CL_InitCGame` (loads VM, calls `CG_INIT`)
- **Per-frame:** `CL_SetCGameTime` (time sync) → `CL_CGameRendering` (triggers `CG_DRAW_ACTIVE_FRAME` inside the VM)
- **Shutdown:** `CL_ShutdownCGame` called on disconnect or map change
- Cgame VM calls back synchronously via `CL_CgameSystemCalls` during any `VM_Call`

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`
- `botlib.h` — `botlib_export_t *botlib_export` (defined in `be_interface.c`)
- `cgvm` — `vm_t *` defined in `cl_main.c`
- `re` — `refexport_t` renderer interface (defined in `cl_main.c`)
- Camera functions (`loadCamera`, `startCamera`, `getCameraInfo`) — declared extern, all call sites commented out
- `CM_*`, `S_*`, `FS_*`, `Key_*`, `CIN_*`, `Cbuf_*`, `Cvar_*`, `Cmd_*`, `Hunk_*`, `Sys_*`, `Com_*` — all defined elsewhere in engine subsystems
