# code/ui/ui_public.h

## File Purpose
Defines the public ABI contract between the Quake III Arena engine and the UI dynamic module (VM). It enumerates all syscall trap numbers the UI module uses to call into the engine (`uiImport_t`) and all entry points the engine calls on the UI module (`uiExport_t`).

## Core Responsibilities
- Declare the UI API version constant for compatibility checks
- Define `uiClientState_t` to carry connection/server state to the UI
- Enumerate all engine→UI import syscalls (`uiImport_t`)
- Enumerate all UI→engine export entry points (`uiExport_t`)
- Define `uiMenuCommand_t` for identifying which menu to activate
- Define server-list sort-order constants

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `uiClientState_t` | struct | Snapshot of client connection state passed to the UI (connection status, server name, messages) |
| `uiImport_t` | enum | Ordered trap numbers for every engine service the UI VM may call via `trap_*` syscall wrappers |
| `uiExport_t` | enum | Ordered entry-point indices for functions the engine calls into the UI module |
| `uiMenuCommand_t` | enum | Identifies which top-level menu the engine requests to activate |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining types and constants only.

## Control Flow Notes
- At load time, the engine queries `UI_GETAPIVERSION` (index 0 of `uiExport_t`) and compares the returned value against `UI_API_VERSION` (6) to validate ABI compatibility.
- During gameplay the engine drives the UI module by dispatching into `uiExport_t` ordinals: `UI_INIT` on startup, `UI_REFRESH` each frame, `UI_KEY_EVENT`/`UI_MOUSE_EVENT` on input, `UI_SET_ACTIVE_MENU` on state transitions, and `UI_SHUTDOWN` on teardown.
- The UI module calls back into the engine by issuing `uiImport_t` trap numbers through its internal `syscall()` mechanism; these cover rendering (`UI_R_*`), sound (`UI_S_*`), cvars (`UI_CVAR_*`), filesystem (`UI_FS_*`), LAN/server-browser (`UI_LAN_*`), input (`UI_KEY_*`), and cinematic (`UI_CIN_*`) services.
- The gap at index 100 (`UI_MEMSET`–`UI_CEIL`) reserves a separate high-numbered range for math/memory builtins, isolating them from the main syscall table.

## External Dependencies
- `connstate_t` — defined in engine connection-state headers (e.g., `client.h`)
- `MAX_STRING_CHARS` — defined in `q_shared.h`
- No includes are present in this header; consumers must include prerequisite headers before this file
