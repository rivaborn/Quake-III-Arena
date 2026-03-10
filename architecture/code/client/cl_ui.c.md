# code/client/cl_ui.c

## File Purpose
This file implements the client-side UI virtual machine (VM) bridge layer, providing the system call dispatch table that translates UI module requests into engine function calls. It also manages the UI VM lifecycle (init/shutdown) and maintains the server browser (LAN) data structures with cache persistence.

## Core Responsibilities
- Dispatch all `UI_*` system calls from the UI VM to engine subsystems via `CL_UISystemCalls`
- Initialize and shut down the UI VM (`CL_InitUI`, `CL_ShutdownUI`)
- Provide LAN server list management: add, remove, query, compare, and mark visibility across four server sources (local, mplayer, global, favorites)
- Persist and restore server browser caches to/from `servercache.dat`
- Bridge UI requests to renderer (`re.*`), sound (`S_*`), key system, filesystem, cinematic, and botlib parse contexts
- Expose client/connection state (`GetClientState`, `CL_GetGlconfig`) and config strings to the UI VM

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `serverInfo_t` | struct (defined in `client.h`) | Holds host name, map, game, ping, client counts, netadr for one server entry |
| `uiClientState_t` | struct (defined externally) | Snapshot of connection state passed to the UI VM |
| `vmInterpret_t` | typedef/enum (defined externally) | Selects VM execution mode (interpreted vs. compiled) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `uivm` | `vm_t *` | global | Handle to the active UI virtual machine instance |
| `botlib_export` | `botlib_export_t *` | extern global | Access to botlib PC (parser) functions used by UI script loading |

## Key Functions / Methods

### CL_UISystemCalls
- **Signature:** `int CL_UISystemCalls( int *args )`
- **Purpose:** Central dispatch for every syscall the UI VM issues; maps `UI_*` enum values to engine calls.
- **Inputs:** `args` — array where `args[0]` is the syscall number; subsequent elements are arguments (integers or float-reinterpreted).
- **Outputs/Return:** Integer result (0 or meaningful value) returned to the VM caller.
- **Side effects:** May modify cvars, filesystem, renderer state, key catchers, sound, server lists, CD key buffer, or trigger errors.
- **Calls:** Nearly all engine subsystems: `Com_Error`, `Cvar_*`, `FS_*`, `re.*`, `S_*`, `Key_*`, `CIN_*`, `botlib_export->PC_*`, `SCR_UpdateScreen`, `Hunk_MemoryRemaining`, `CL_CDKeyValidate`, and all local `LAN_*` helpers.
- **Notes:** Uses `VMA(x)` / `VMF(x)` macros to extract typed VM arguments via `VM_ArgPtr`. Unrecognized syscall triggers `ERR_DROP`.

### CL_InitUI
- **Signature:** `void CL_InitUI( void )`
- **Purpose:** Creates the UI VM, verifies its API version, and calls `UI_INIT` to start the UI for the current connection state.
- **Inputs:** None (reads `cl_connectedToPureServer`, `vm_ui` cvar).
- **Outputs/Return:** None.
- **Side effects:** Allocates `uivm`; calls `VM_Create` and `VM_Call(UI_INIT)`; sets `cls.uiStarted = qfalse` on version mismatch; fatal error if VM creation fails.
- **Calls:** `VM_Create`, `VM_Call`, `Cvar_VariableValue`, `Com_Error`.
- **Notes:** Accepts the legacy `UI_OLD_API_VERSION` (4) silently; any other mismatch is a drop error.

### CL_ShutdownUI
- **Signature:** `void CL_ShutdownUI( void )`
- **Purpose:** Tears down the UI VM cleanly.
- **Side effects:** Calls `UI_SHUTDOWN` on the VM, frees it, sets `uivm = NULL`, clears `KEYCATCH_UI` and `cls.uiStarted`.
- **Calls:** `VM_Call(UI_SHUTDOWN)`, `VM_Free`.

### LAN_AddServer
- **Signature:** `static int LAN_AddServer(int source, const char *name, const char *address)`
- **Purpose:** Inserts a new server entry into the appropriate list if not already present and list not full.
- **Outputs/Return:** `1` = added, `0` = duplicate, `-1` = list full or invalid source.
- **Side effects:** Modifies `cls.*Servers` arrays and count fields.
- **Calls:** `NET_StringToAdr`, `NET_CompareAdr`, `Q_strncpyz`.

### LAN_LoadCachedServers / LAN_SaveServersToCache
- **Signature:** `void LAN_LoadCachedServers()` / `void LAN_SaveServersToCache()`
- **Purpose:** Read/write the server browser cache (`servercache.dat`) in the SV filesystem namespace.
- **Side effects:** Modifies `cls.numglobalservers`, `cls.nummplayerservers`, `cls.numfavoriteservers`, `cls.numGlobalServerAddresses`, and the three server arrays. On load, validates blob size before accepting data.
- **Calls:** `FS_SV_FOpenFileRead`, `FS_SV_FOpenFileWrite`, `FS_Read`, `FS_Write`, `FS_FCloseFile`.

### LAN_CompareServers
- **Signature:** `static int LAN_CompareServers(int source, int sortKey, int sortDir, int s1, int s2)`
- **Purpose:** Comparator for UI-driven server list sorting; supports host, map, clients, game type, and ping keys with direction flag.
- **Outputs/Return:** `<0`, `0`, or `>0` (negated when `sortDir` is non-zero).

### GetConfigString
- **Signature:** `static int GetConfigString(int index, char *buf, int size)`
- **Purpose:** Retrieves a gamestate config string by index from `cl.gameState`.
- **Outputs/Return:** `qtrue` if string found and copied; `qfalse` on invalid index or missing offset.

### Notes on minor helpers
- `FloatAsInt` — type-puns `float` to `int` for returning floats through the integer VM return channel.
- `GetClientState`, `CL_GetGlconfig`, `GetClipboardData`, `Key_KeynumToStringBuf`, `Key_GetBindingBuf`, `Key_GetCatcher/SetCatcher`, `CLUI_GetCDKey/SetCDKey` — thin wrappers copying data between engine globals and UI-facing buffers.
- `LAN_ResetPings`, `LAN_RemoveServer`, `LAN_GetServerCount`, `LAN_GetServerAddressString`, `LAN_GetServerInfo`, `LAN_GetServerPing`, `LAN_GetServerPtr`, `LAN_MarkServerVisible`, `LAN_ServerIsVisible` — accessors/mutators over the four server arrays in `cls`.
- `LAN_GetPingQueueCount`, `LAN_ClearPing`, `LAN_GetPing`, `LAN_GetPingInfo`, `LAN_UpdateVisiblePings`, `LAN_GetServerStatus` — delegate directly to `CL_*` counterparts.
- `UI_usesUniqueCDKey`, `UI_GameCommand` — call `VM_Call` on `uivm` if active, safe-guard against null.

## Control Flow Notes
- **Init:** `CL_InitUI` is called from `CL_StartHunkUsers` during client startup or map load.
- **Per-frame:** `UI_GameCommand` is polled each frame to let the UI claim console commands; the UI VM is driven by `VM_Call(UI_REFRESH)` elsewhere (not in this file).
- **Syscall path:** Every UI VM trap funnels through `CL_UISystemCalls` synchronously during VM execution.
- **Shutdown:** `CL_ShutdownUI` is called on disconnect or full client shutdown.

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`), `../game/botlib.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `cl` (`clientActive_t`), `re` (`refexport_t`), `cl_connectedToPureServer`, `cl_cdkey`, `cvar_modifiedFlags`, `VM_Create/Call/Free/ArgPtr`, `NET_*`, `FS_*`, `S_*`, `Key_*`, `CIN_*`, `SCR_UpdateScreen`, `Sys_GetClipboardData`, `Sys_Milliseconds`, `Hunk_MemoryRemaining`, `Com_RealTime`, `CL_CDKeyValidate`, `CL_ServerStatus`, `CL_UpdateVisiblePings_f`, `CL_GetPing*`, `Z_Free`
