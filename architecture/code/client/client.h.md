# code/client/client.h

## File Purpose
Primary header for the Quake III Arena client subsystem. Defines the three core client state structures (`clSnapshot_t`, `clientActive_t`, `clientConnection_t`, `clientStatic_t`) and declares all function prototypes for every client-side module: main, input, parsing, console, screen, cinematics, cgame, UI, and network channel.

## Core Responsibilities
- Defines snapshot representation for server-to-client delta-compressed state
- Defines the three-tier client state hierarchy (active/connection/static)
- Declares all inter-module function interfaces for the client subsystem
- Declares the global extern instances (`cl`, `clc`, `cls`) shared across client modules
- Declares VM handles for cgame and UI modules (`cgvm`, `uivm`, `re`)
- Declares all client-facing cvars as extern pointers
- Pulls in all required subsystem headers (renderer, UI, sound, cgame, shared)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `clSnapshot_t` | struct | One server-state snapshot: validity, server time, player state, entity count, area visibility mask |
| `outPacket_t` | struct | Tracks timing metadata for each outgoing client packet (for ping/delta correlation) |
| `clientActive_t` | struct | Per-gamestate client data: snapshots, entity baselines, parse ring buffer, user commands, view angles |
| `clientConnection_t` | struct | Per-connection state: netchan, reliable command queues, demo playback/record, file downloads |
| `ping_t` | struct | A single in-flight ping request with address, start time, and result |
| `serverInfo_t` | struct | Server browser entry: hostname, map, game, ping, player counts |
| `serverAddress_t` | struct | Compact IP+port for storing global server address lists |
| `clientStatic_t` | struct | Persistent client state (survives disconnects): connection state, subsystem flags, server browser lists, render config |
| `kbutton_t` | struct | Input button state tracking two simultaneous key-down holders, duration, and press flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cl` | `clientActive_t` | global | Active per-gamestate client data |
| `clc` | `clientConnection_t` | global | Active connection state |
| `cls` | `clientStatic_t` | global | Persistent client state |
| `cgvm` | `vm_t *` | global | Handle to the cgame VM/DLL |
| `uivm` | `vm_t *` | global | Handle to the UI VM/DLL |
| `re` | `refexport_t` | global | Renderer export function table |
| `g_console_field_width` | `int` | global | Width of console input field in characters |
| `cl_connectedToPureServer` | `int` | global | Non-zero if connected to a pure (pak-verified) server |
| `in_mlook`, `in_klook`, `in_strafe`, `in_speed` | `kbutton_t` | global | Core input button states for mouse/keyboard look and movement |

## Key Functions / Methods

Functions are declared here but defined in their respective `.c` files; only signatures are present.

### CL_Init
- Signature: `void CL_Init(void)`
- Purpose: Initializes the entire client subsystem
- Inputs: None
- Outputs/Return: void
- Side effects: Registers cvars, commands; sets up subsystem state in `cls`
- Calls: Defined in `cl_main.c`
- Notes: Called once at engine startup

### CL_Frame (declared in qcommon.h, driven by this header's state)
- Not declared here directly; client frame is driven by `Com_Frame` → `CL_Frame`.

### CL_ParseServerMessage
- Signature: `void CL_ParseServerMessage(msg_t *msg)`
- Purpose: Processes an incoming server message, dispatching on `svc_ops_e` opcodes
- Inputs: `msg` — incoming bitstream message
- Outputs/Return: void
- Side effects: Updates `cl` snapshot/entity state; may trigger `CL_SystemInfoChanged`
- Calls: Defined in `cl_parse.c`

### CL_WritePacket
- Signature: `void CL_WritePacket(void)`
- Purpose: Builds and sends a client movement packet to the server
- Inputs: None (reads from `cl.cmds`, `clc`)
- Outputs/Return: void
- Side effects: Transmits via `clc.netchan`; updates `cl.outPackets`

### CL_SendCmd
- Signature: `void CL_SendCmd(void)`
- Purpose: Called each client frame to build a usercmd and invoke `CL_WritePacket`
- Inputs: None
- Outputs/Return: void
- Side effects: Modifies `cl.cmdNumber`, `cl.cmds`

### SCR_UpdateScreen
- Signature: `void SCR_UpdateScreen(void)`
- Purpose: Top-level screen refresh; orchestrates 2D UI and 3D scene rendering each frame
- Inputs: None
- Outputs/Return: void
- Side effects: Calls into `re` (renderer), cgame, UI draw functions

### CL_Netchan_Transmit / CL_Netchan_Process
- Signature: `void CL_Netchan_Transmit(netchan_t *chan, msg_t *msg)` / `qboolean CL_Netchan_Process(netchan_t *chan, msg_t *msg)`
- Purpose: Client-side wrappers around `Netchan_Transmit`/`Netchan_Process` that apply Huffman encoding/decoding
- Side effects: I/O — transmits UDP packets; modifies message buffer on receive

### Notes
- Trivial getter/setter wrappers (`Key_GetCatcher`, `Key_SetCatcher`, `CL_GetPing`, `CL_ClearPing`) are declared but not individually documented above.
- Cinematic functions (`CIN_PlayCinematic`, `CIN_RunCinematic`, etc.) form a self-contained group in `cl_cin.c`.

## Control Flow Notes
- `clientStatic_t cls` is never zeroed; it persists across map loads and disconnects.
- `clientActive_t cl` is wiped on every new `gameState_t` (map load / map_restart).
- `clientConnection_t clc` is wiped on disconnect.
- Each engine frame: `Com_Frame` → `CL_Frame` → `CL_SendCmd` + `SCR_UpdateScreen`; incoming packets arrive via `CL_ReadPackets` → `CL_ParseServerMessage`, updating `cl.snapshots[]` and `cl.parseEntities[]`.

## External Dependencies
- `../game/q_shared.h` — shared math, entity/player state types, `qboolean`, `cvar_t`
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, filesystem, `netadr_t`, `connstate_t`
- `../renderer/tr_public.h` — `refexport_t`, `glconfig_t`, `stereoFrame_t`
- `../ui/ui_public.h` — `uiClientState_t`, `uiMenuCommand_t`
- `keys.h` — `qkey_t`, key binding declarations, `field_t` input fields
- `snd_public.h` — sound system public interface (included but not shown)
- `../cgame/cg_public.h` — cgame public interface, `stereoFrame_t`
- `../game/bg_public.h` — `playerState_t`, `usercmd_t`, pmove shared definitions
- **Defined elsewhere:** `vm_t`, `gameState_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `netchan_t`, all `Netchan_*` base functions, `MSG_*` functions
