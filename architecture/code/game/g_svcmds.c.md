# code/game/g_svcmds.c

## File Purpose
Implements server-console-only commands for the Quake III Arena game module, including IP-based packet filtering/banning and administrative commands such as entity listing, team forcing, and bot management dispatch.

## Core Responsibilities
- Maintain an in-memory IP filter list (`ipFilters[]`) for allow/deny packet filtering
- Parse and persist IP ban masks to/from the `g_banIPs` cvar string
- Provide `G_FilterPacket` to gate incoming connections against the filter list
- Expose `Svcmd_AddIP_f` / `Svcmd_RemoveIP_f` for runtime ban management
- Implement `ConsoleCommand` as the single dispatch entry point for all server-console commands
- Provide `ClientForString` helper to resolve a client by slot number or name

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ipFilter_t` | struct | Stores a bitmask (`mask`) and comparison value (`compare`) representing one IP filter entry |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ipFilters` | `ipFilter_t[MAX_IPFILTERS]` | static (file) | Array of up to 1024 active IP filter entries |
| `numIPFilters` | `int` | static (file) | Count of currently allocated filter slots |

## Key Functions / Methods

### StringToFilter
- **Signature:** `static qboolean StringToFilter(char *s, ipFilter_t *f)`
- **Purpose:** Parses a dot-notation IP string (with `*` wildcards) into an `ipFilter_t` mask/compare pair.
- **Inputs:** `s` — dotted IP string (e.g. `"192.168.1.*"`); `f` — output filter struct.
- **Outputs/Return:** `qtrue` on success, `qfalse` on malformed input.
- **Side effects:** Prints error via `G_Printf` on bad input.
- **Calls:** `atoi`, `G_Printf`
- **Notes:** Uses direct `*(unsigned *)` casts for byte-array-to-uint conversion; assumes little-endian byte order implicitly.

### UpdateIPBans
- **Signature:** `static void UpdateIPBans(void)`
- **Purpose:** Serializes the current `ipFilters[]` array back into the `g_banIPs` cvar as a space-separated list of dotted IP masks.
- **Inputs:** None (reads `ipFilters`, `numIPFilters`).
- **Outputs/Return:** void
- **Side effects:** Calls `trap_Cvar_Set("g_banIPs", ...)`, modifying persistent cvar state. Prints overflow warning to `Com_Printf` if the string exceeds `MAX_CVAR_VALUE_STRING`.
- **Calls:** `Q_strcat`, `va`, `strlen`, `trap_Cvar_Set`, `Com_Printf`

### G_FilterPacket
- **Signature:** `qboolean G_FilterPacket(char *from)`
- **Purpose:** Tests a client's IP address string against all active filters; returns whether the packet should be blocked.
- **Inputs:** `from` — IP:port string of the connecting client.
- **Outputs/Return:** `qtrue` if the packet should be filtered out per `g_filterBan` mode; `qfalse` otherwise.
- **Side effects:** None.
- **Calls:** None (inline parsing loop).
- **Notes:** `g_filterBan == 1` (default) = ban list; `g_filterBan == 0` = allow-list. A match returns `g_filterBan.integer != 0`; no match returns `g_filterBan.integer == 0`.

### G_ProcessIPBans
- **Signature:** `void G_ProcessIPBans(void)`
- **Purpose:** Reads the `g_banIPs` cvar string on startup and populates `ipFilters[]` by calling `AddIP` for each space-delimited token.
- **Inputs:** None (reads `g_banIPs.string`).
- **Side effects:** Modifies `ipFilters[]`, `numIPFilters`, and calls `UpdateIPBans` (via `AddIP`).
- **Calls:** `Q_strncpyz`, `strchr`, `AddIP`

### AddIP *(static)*
- **Signature:** `static void AddIP(char *str)`
- **Purpose:** Adds one IP mask to `ipFilters[]`, reusing slots marked `0xffffffff`, then syncs to cvar.
- **Calls:** `StringToFilter`, `UpdateIPBans`, `G_Printf`
- **Notes:** Entries that fail parsing are stored as `0xffffffff` (sentinel for "free slot").

### ClientForString
- **Signature:** `gclient_t *ClientForString(const char *s)`
- **Purpose:** Resolves a client by numeric slot index or by case-insensitive name match.
- **Inputs:** `s` — slot number string or player name.
- **Outputs/Return:** Pointer to matching `gclient_t`, or `NULL` if not found/disconnected.
- **Calls:** `atoi`, `Q_stricmp`, `G_Printf`, `Com_Printf`

### ConsoleCommand
- **Signature:** `qboolean ConsoleCommand(void)`
- **Purpose:** Entry point called by the server for every server-console command. Dispatches to the appropriate handler by string comparison.
- **Inputs:** None directly — reads argv[0] via `trap_Argv`.
- **Outputs/Return:** `qtrue` if the command was handled, `qfalse` to pass it along.
- **Side effects:** May invoke any of the `Svcmd_*` functions or `trap_SendServerCommand` / `trap_SendConsoleCommand`.
- **Calls:** `trap_Argv`, `Q_stricmp`, `Svcmd_EntityList_f`, `Svcmd_ForceTeam_f`, `Svcmd_GameMem_f`, `Svcmd_AddBot_f`, `Svcmd_BotList_f`, `Svcmd_AbortPodium_f`, `Svcmd_AddIP_f`, `Svcmd_RemoveIP_f`, `trap_SendConsoleCommand`, `trap_SendServerCommand`, `ConcatArgs`
- **Notes:** On a dedicated server, unrecognized commands are echoed as chat from "server".

## Control Flow Notes
- `G_ProcessIPBans` is called during game init (from `g_main.c`) to restore persisted bans.
- `G_FilterPacket` is called from `ClientConnect` (g_client.c) to reject banned IPs before a client is admitted.
- `ConsoleCommand` is called each frame/event by the server engine when a console command is issued; it is the sole dispatch point for this file's functionality at runtime.

## External Dependencies
- **Includes:** `g_local.h` (which transitively brings in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Argv`, `trap_Argc`, `trap_Cvar_Set`, `trap_SendConsoleCommand`, `trap_SendServerCommand` — VM syscall stubs (`g_syscalls.c`)
  - `G_Printf`, `Com_Printf` — logging (`g_main.c` / engine)
  - `SetTeam` — `g_cmds.c`
  - `ConcatArgs` — `g_cmds.c` (declared but not defined here)
  - `Svcmd_GameMem_f` — `g_mem.c`; `Svcmd_AddBot_f`, `Svcmd_BotList_f` — `g_bot.c`; `Svcmd_AbortPodium_f` — `g_arenas.c`
  - `g_filterBan`, `g_banIPs`, `g_dedicated` — cvars declared in `g_local.h`, registered in `g_main.c`
  - `level`, `g_entities` — global game state (`g_main.c`)
