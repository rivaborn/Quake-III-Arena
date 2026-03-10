# code/client/cl_main.c

## File Purpose
This is the central client subsystem manager for Quake III Arena, responsible for initializing, running, and shutting down all client-side systems. It drives the per-frame client loop, manages the connection state machine (connecting → challenging → connected → active), and owns demo recording/playback, server discovery, and reliable command queuing.

## Core Responsibilities
- Register and manage all client-side cvars and console commands
- Drive the per-frame `CL_Frame` loop: input, timeout, packet send, screen/audio/cinematic update
- Manage connection lifecycle: connect, disconnect, reconnect, challenge/authorize handshake
- Record and play back demo files (write gamestate snapshot + replayed net messages)
- Handle out-of-band connectionless packets (challenge, MOTD, server status, server list)
- Manage file downloads from server (download queue, temp files, FS restart on completion)
- Initialize and teardown renderer, sound, UI, and cgame subsystems via hunk lifecycle
- Provide server browser ping infrastructure (`cl_pinglist`, `CL_UpdateVisiblePings_f`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `clientActive_t` | struct (typedef, defined in client.h) | Per-connection game state: snapshots, entity baselines, cmds, viewangles |
| `clientConnection_t` | struct (typedef, defined in client.h) | Network/demo connection state: reliable cmds, netchan, download state |
| `clientStatic_t` | struct (typedef, defined in client.h) | Persistent client state across connections: render config, server lists, timing |
| `serverStatus_t` | struct | Tracks a pending/completed `getstatus` request: string, address, timing, flags |
| `ping_t` | struct (defined in client.h) | Single ping-request slot: address, start time, result time, info string |
| `serverInfo_t` | struct (defined in client.h) | Cached server browser entry: hostname, map, ping, client counts |
| `serverAddress_t` | struct (defined in client.h) | Compact IP+port storage for overflow global server list |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cl` | `clientActive_t` | global | Active per-level client state |
| `clc` | `clientConnection_t` | global | Current connection/demo state |
| `cls` | `clientStatic_t` | global | Persistent client state (never wiped) |
| `cgvm` | `vm_t *` | global | VM handle for cgame module |
| `re` | `refexport_t` | global | Renderer function table populated by `GetRefAPI` |
| `cl_pinglist` | `ping_t[MAX_PINGREQUESTS]` | global | Active ping request slots for server browser |
| `cl_serverStatusList` | `serverStatus_t[MAX_SERVERSTATUSREQUESTS]` | static/global | Pending server status (`getstatus`) responses |
| `serverStatusCount` | `int` | global | Rolling counter for allocating status slots |
| `demoName` | `char[MAX_QPATH]` | file-static | Workaround for compiler bug in `CL_Record_f` |

Numerous `cvar_t *` globals (e.g., `cl_timeout`, `cl_maxpackets`, `m_pitch`, etc.) are module-level and declared `extern` in `client.h`.

## Key Functions / Methods

### CL_Init
- **Signature:** `void CL_Init(void)`
- **Purpose:** Full client subsystem initialization — registers cvars, console commands, renderer, and screen.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Populates all `cvar_t *` globals; registers ~25 console commands; calls `CL_InitRef`, `SCR_Init`, `Cbuf_Execute`; sets `cl_running = 1`.
- **Calls:** `Con_Init`, `CL_ClearState`, `CL_InitInput`, `Cvar_Get` (×many), `Cmd_AddCommand` (×many), `CL_InitRef`, `SCR_Init`, `Cbuf_Execute`
- **Notes:** Must be called once at engine startup before any client operation.

### CL_Shutdown
- **Signature:** `void CL_Shutdown(void)`
- **Purpose:** Tears down all client subsystems; guards against recursive calls.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `CL_Disconnect`, `S_Shutdown`, `CL_ShutdownRef`, `CL_ShutdownUI`; removes all registered commands; zeroes `cls`; sets `cl_running = 0`.
- **Calls:** `CL_Disconnect`, `S_Shutdown`, `CL_ShutdownRef`, `CL_ShutdownUI`, `Cmd_RemoveCommand` (×many)
- **Notes:** Static `recursive` flag prevents re-entry from error handlers.

### CL_Frame
- **Signature:** `void CL_Frame(int msec)`
- **Purpose:** Per-frame driver for the entire client subsystem; called from the main engine loop.
- **Inputs:** `msec` — milliseconds since last frame
- **Outputs/Return:** None
- **Side effects:** Advances `cls.realtime`; may trigger UI menus; calls screen, audio, cinematic, console updates; sends commands and checks connection timeout.
- **Calls:** `CL_CheckUserinfo`, `CL_CheckTimeout`, `CL_SendCmd`, `CL_CheckForResend`, `CL_SetCGameTime`, `SCR_UpdateScreen`, `S_Update`, `SCR_RunCinematic`, `Con_RunConsole`
- **Notes:** AVI demo mode locks `msec` to a fixed rate. Returns early if `com_cl_running` is false.

### CL_Disconnect
- **Signature:** `void CL_Disconnect(qboolean showMainMenu)`
- **Purpose:** Terminates the current connection or demo; transitions to disconnected state.
- **Inputs:** `showMainMenu` — if true, calls UI to show main menu.
- **Outputs/Return:** None
- **Side effects:** Stops demo recording, closes download/demo files, sends 3× reliable disconnect packets, zeroes `clc`, sets `cls.state = CA_DISCONNECTED`.
- **Calls:** `CL_StopRecord_f`, `FS_FCloseFile`, `VM_Call`, `SCR_StopCinematic`, `S_ClearSoundBuffer`, `CL_AddReliableCommand`, `CL_WritePacket`, `CL_ClearState`
- **Notes:** Safe to call from `Com_Error` and `Com_Quit`.

### CL_AddReliableCommand
- **Signature:** `void CL_AddReliableCommand(const char *cmd)`
- **Purpose:** Enqueues a command guaranteed to reach the server in order.
- **Inputs:** `cmd` — null-terminated command string
- **Outputs/Return:** None
- **Side effects:** Increments `clc.reliableSequence`; writes into `clc.reliableCommands` ring buffer.
- **Calls:** `Com_Error` (on overflow), `Q_strncpyz`
- **Notes:** Calls `Com_Error(ERR_DROP)` if the buffer overflows (unacknowledged window exceeded).

### CL_Record_f
- **Signature:** `void CL_Record_f(void)`
- **Purpose:** Starts demo recording; writes a synthetic gamestate packet as the demo header.
- **Inputs:** Console argument — optional demo filename.
- **Outputs/Return:** None
- **Side effects:** Opens a file in `demos/`; sets `clc.demorecording = qtrue`; writes configstrings and entity baselines into the file.
- **Calls:** `FS_FOpenFileWrite`, `MSG_Init`, `MSG_WriteByte/Long/Short/BigString/DeltaEntity`, `FS_Write`
- **Notes:** If no name given, auto-scans `demo0000`–`demo9999` for a free slot.

### CL_ReadDemoMessage
- **Signature:** `void CL_ReadDemoMessage(void)`
- **Purpose:** Reads one packet from the demo file and feeds it to the server message parser.
- **Inputs:** None (reads from `clc.demofile`)
- **Outputs/Return:** None
- **Side effects:** Updates `clc.serverMessageSequence`; calls `CL_ParseServerMessage`; calls `CL_DemoCompleted` on EOF or error.
- **Calls:** `FS_Read`, `MSG_Init`, `CL_ParseServerMessage`, `CL_DemoCompleted`

### CL_PacketEvent
- **Signature:** `void CL_PacketEvent(netadr_t from, msg_t *msg)`
- **Purpose:** Entry point for all incoming UDP packets from the main event loop.
- **Inputs:** `from` — sender address; `msg` — raw packet buffer.
- **Outputs/Return:** None
- **Side effects:** Routes to `CL_ConnectionlessPacket` or `CL_ParseServerMessage`; optionally records to demo via `CL_WriteDemoMessage`.
- **Calls:** `CL_ConnectionlessPacket`, `CL_Netchan_Process`, `CL_ParseServerMessage`, `CL_WriteDemoMessage`

### CL_ConnectionlessPacket
- **Signature:** `void CL_ConnectionlessPacket(netadr_t from, msg_t *msg)`
- **Purpose:** Dispatches out-of-band packets by command string token.
- **Inputs:** `from`, `msg`
- **Outputs/Return:** None
- **Side effects:** May advance connection state, update server lists, respond to MOTD, etc.
- **Calls:** `CL_ServerInfoPacket`, `CL_ServerStatusResponse`, `CL_DisconnectPacket`, `CL_MotdPacket`, `CL_ServersResponsePacket`, `Netchan_Setup`

### CL_CheckForResend
- **Signature:** `void CL_CheckForResend(void)`
- **Purpose:** Retransmits connection handshake packets if no reply has arrived within `RETRANSMIT_TIMEOUT`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sends `getchallenge` (CA_CONNECTING) or `connect` (CA_CHALLENGING) OOB packets; may call `CL_RequestAuthorization`.
- **Notes:** Called every frame from `CL_Frame`.

### CL_InitRef / CL_ShutdownRef
- **Signature:** `void CL_InitRef(void)` / `void CL_ShutdownRef(void)`
- **Purpose:** Initialize/teardown the renderer DLL/module; populate the `re` function table.
- **Side effects:** `CL_InitRef` calls `GetRefAPI` and fills `re`; `CL_ShutdownRef` calls `re.Shutdown(qtrue)` and zeroes `re`.

### CL_StartHunkUsers
- **Signature:** `void CL_StartHunkUsers(void)`
- **Purpose:** (Re)starts renderer, sound, and UI after a hunk clear.
- **Calls:** `CL_InitRenderer`, `S_Init`, `S_BeginRegistration`, `CL_InitUI`
- **Notes:** Guards each subsystem with a `started` flag to avoid double-init.

### CL_FlushMemory
- **Signature:** `void CL_FlushMemory(void)`
- **Purpose:** Shuts down all client subsystems, clears hunk memory, then restarts them.
- **Calls:** `CL_ShutdownAll`, `Hunk_Clear` or `Hunk_ClearToMark`, `CM_ClearMap`, `CL_StartHunkUsers`

## Control Flow Notes
- **Init:** `CL_Init` called once at engine startup.
- **Per-frame:** `CL_Frame(msec)` is the client's frame entry point, invoked from `Com_Frame` in `common.c`.
- **Network events:** Packets arrive via `CL_PacketEvent` (called from `Com_EventLoop`).
- **Map load:** `CL_MapLoading` → `CL_FlushMemory` → `CL_StartHunkUsers` → `CL_InitCGame`.
- **Shutdown:** `CL_Shutdown` called once at engine exit.

## External Dependencies
- **Includes:** `client.h` (aggregates `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`); `<limits.h>`
- **Defined elsewhere:**
  - `GetRefAPI` — renderer module export
  - `SV_BotFrame`, `SV_Shutdown`, `SV_Frame` — server module
  - `CL_ParseServerMessage` — `cl_parse.c`
  - `CL_SendCmd`, `CL_WritePacket`, `CL_InitInput` — `cl_input.c`
  - `CL_InitCGame`, `CL_ShutdownCGame`, `CL_SetCGameTime` — `cl_cgame.c`
  - `CL_InitUI`, `CL_ShutdownUI` — `cl_ui.c`
  - `CL_Netchan_Process` — `cl_net_chan.c`
  - `S_Init`, `S_Shutdown`, `S_Update`, `S_DisableSounds`, etc. — sound subsystem
  - `Hunk_*`, `CM_*`, `FS_*`, `NET_*`, `Cvar_*`, `Cmd_*`, `MSG_*` — `qcommon`
