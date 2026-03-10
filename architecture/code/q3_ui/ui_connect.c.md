# code/q3_ui/ui_connect.c

## File Purpose
Renders the connection/loading screen shown while the client connects to a server. Handles display of connection state transitions, active file download progress, and ESC-key disconnection.

## Core Responsibilities
- Draw the full-screen connection overlay (background, server name, map name, MOTD)
- Display per-state status text (challenging, connecting, awaiting gamestate)
- Show real-time download progress: file size, transfer rate, estimated time remaining
- Track the last connection state to reset loading text on regression
- Handle the ESC key during connection to issue a disconnect command

## Key Types / Data Structures
None defined in this file; relies on `uiClientState_t`, `connstate_t`, and `menufield_s` from headers.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `passwordNeeded` | `qboolean` | global | Flag indicating a server password is required (unused/dead code, always `qtrue`) |
| `passwordField` | `menufield_s` | global | UI field for entering a server password (declared but never rendered; guarded by `#if 0`) |
| `lastConnState` | `connstate_t` | static | Tracks previous connection state to detect regressions and clear loading text |
| `lastLoadingText` | `char[MAX_INFO_VALUE]` | static | Stores the last displayed loading string (declared but not actively used for comparison) |

## Key Functions / Methods

### UI_ReadableSize
- Signature: `static void UI_ReadableSize(char *buf, int bufsize, int value)`
- Purpose: Formats a byte count into a human-readable string (GB/MB/KB/bytes).
- Inputs: Output buffer, buffer size, raw byte value.
- Outputs/Return: Writes formatted string into `buf`.
- Side effects: None.
- Calls: `Com_sprintf`, `strlen`.
- Notes: Uses integer arithmetic; GB/MB branches include a two-decimal fraction via modulo.

### UI_PrintTime
- Signature: `static void UI_PrintTime(char *buf, int bufsize, int time)`
- Purpose: Formats a millisecond duration into `hr/min/sec` human-readable string.
- Inputs: Output buffer, buffer size, time in milliseconds.
- Outputs/Return: Writes into `buf`.
- Side effects: None.
- Calls: `Com_sprintf`.

### UI_DisplayDownloadInfo
- Signature: `static void UI_DisplayDownloadInfo(const char *downloadName)`
- Purpose: Renders download progress UI: file name, percent complete, ETA, transfer rate, bytes copied.
- Inputs: The current download filename.
- Outputs/Return: Void; draws directly to screen.
- Side effects: Reads cvars `cl_downloadSize`, `cl_downloadCount`, `cl_downloadTime`; reads `uis.realtime`.
- Calls: `trap_Cvar_VariableValue`, `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`, `UI_DrawProportionalString`, `UI_ReadableSize`, `UI_PrintTime`, `va`.
- Notes: Guards against division-by-zero when elapsed time is zero. ETA computed in integer KB to avoid 32-bit overflow around 4 MB. Contains several commented-out `fprintf` debug lines from bk010104/bk010108.

### UI_DrawConnectScreen
- Signature: `void UI_DrawConnectScreen(qboolean overlay)`
- Purpose: Main entry point called each frame during connection; draws background, server/map info, MOTD, connection state string, or delegates to download UI.
- Inputs: `overlay` — if `qtrue`, skips drawing the background (used when overlaid on cgame loading screen).
- Outputs/Return: Void; renders to screen.
- Side effects: Calls `Menu_Cache` (loads cached shaders); reads client state via `trap_GetClientState`; reads `CS_SERVERINFO` config string; updates `lastConnState`.
- Calls: `Menu_Cache`, `UI_SetColor`, `UI_DrawHandlePic`, `trap_GetClientState`, `trap_GetConfigString`, `Info_ValueForKey`, `UI_DrawProportionalString`, `UI_DrawProportionalString_AutoWrapped`, `UI_DisplayDownloadInfo`, `va`.
- Notes: `CA_LOADING` and `CA_PRIMED` states return early without drawing state text. Password field block is completely dead (`#if 0`). On state regression (`lastConnState > cstate.connState`), clears `lastLoadingText`.

### UI_KeyConnect
- Signature: `void UI_KeyConnect(int key)`
- Purpose: Handles key input on the connect screen; only ESC is handled, issuing a `disconnect` command.
- Inputs: `key` — key code.
- Outputs/Return: Void.
- Side effects: Appends `disconnect\n` to the command buffer via `trap_Cmd_ExecuteText`.
- Calls: `trap_Cmd_ExecuteText`.

## Control Flow Notes
`UI_DrawConnectScreen` is called every UI refresh frame while the client is in a connecting state. It is also called from the cgame as an overlay (`overlay=qtrue`) during map loading to prevent the screen blinking. `UI_KeyConnect` is the key handler registered for this screen; it is not itself responsible for routing, which is managed by `ui_atoms.c`/`ui_main.c`.

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`, `Menu_Cache`, `UI_SetColor`, `UI_DrawHandlePic`, `UI_DrawProportionalString`, `UI_DrawProportionalString_AutoWrapped`, `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`, `Info_ValueForKey`, `Com_sprintf`, `va`, `uis` (global `uiStatic_t`)
