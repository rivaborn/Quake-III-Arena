# code/win32/win_local.h

## File Purpose
Win32-platform-specific header for Quake III Arena, declaring the shared Windows application state, input/sound subsystem interfaces, and window procedure used across all Win32 platform modules.

## Core Responsibilities
- Declares the `WinVars_t` struct holding global Win32 application state (window handle, instance, OS version, etc.)
- Declares the input subsystem API (`IN_*` functions)
- Declares the system event queue injection point (`Sys_QueEvent`)
- Declares the Win32 console management functions
- Declares the DirectSound activation and init hooks
- Exports the main window procedure (`MainWndProc`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WinVars_t` | struct | Aggregates all global Win32 state: renderer DLL handle, window handle, app instance, focus/minimize flags, OS version info, and last system message timestamp |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `g_wv` | `WinVars_t` | global (extern) | Singleton holding all Win32 platform state; defined in `win_main.c`, referenced across all Win32 modules |

## Key Functions / Methods

### IN_Init / IN_Shutdown
- Signature: `void IN_Init(void)` / `void IN_Shutdown(void)`
- Purpose: Initialize and tear down the DirectInput-based input subsystem
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/releases DirectInput COM objects; registers mouse/joystick devices
- Calls: Defined elsewhere (`win_input.c`)
- Notes: Must be called after window creation; `IN_Shutdown` must be idempotent

### IN_Frame
- Signature: `void IN_Frame(void)`
- Purpose: Per-frame input polling; reads accumulated mouse/joystick state and queues events
- Inputs: None
- Outputs/Return: None
- Side effects: Calls `Sys_QueEvent` to inject input events into the event queue
- Calls: Defined elsewhere (`win_input.c`)
- Notes: Called once per game frame from the platform main loop

### IN_Move
- Signature: `void IN_Move(usercmd_t *cmd)`
- Purpose: Appends non-keyboard, non-mouse movement (joystick) to a `usercmd_t`
- Inputs: `cmd` — partially-filled user command struct
- Outputs/Return: Mutates `*cmd` in place
- Side effects: None beyond modifying `*cmd`
- Calls: Defined elsewhere

### Sys_QueEvent
- Signature: `void Sys_QueEvent(int time, sysEventType_t type, int value, int value2, int ptrLength, void *ptr)`
- Purpose: Injects a platform event (key, mouse, network, etc.) into the engine's cross-platform event queue
- Inputs: `time` — event timestamp; `type` — event category; `value`/`value2` — event-specific data; `ptrLength`/`ptr` — optional payload
- Outputs/Return: None
- Side effects: Writes to the global sysEvent ring buffer (defined in `common.c`)
- Calls: Defined elsewhere (`win_main.c` / `common.c`)

### MainWndProc
- Signature: `LONG WINAPI MainWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)`
- Purpose: Win32 window procedure; dispatches Windows messages to appropriate engine subsystems
- Inputs: Standard Win32 `WndProc` parameters
- Outputs/Return: `LONG` — message handling result
- Side effects: Updates `g_wv.sysMsgTime`, `g_wv.activeApp`, `g_wv.isMinimized`; calls input and renderer callbacks
- Calls: Defined elsewhere (`win_wndproc.c`)

### Notes
- `Sys_CreateConsole`/`Sys_DestroyConsole`/`Sys_ConsoleInput` manage the dedicated-server text console window
- `IN_MouseEvent`, `IN_Activate`, `IN_DeactivateWin32Mouse`, `IN_JoystickCommands` are narrow helpers called from `MainWndProc` or `IN_Frame`
- `SNDDMA_Activate`/`SNDDMA_InitDS` are DirectSound hooks declared here but implemented in `win_snd.c`

## Control Flow Notes
This header is included by all `code/win32/win_*.c` files. `g_wv` is initialized during `WinMain` startup (`win_main.c`). `IN_Init` is called during renderer/window init; `IN_Frame` is called every frame from the main loop; `MainWndProc` is registered as the `WNDCLASS.lpfnWndProc` and fires asynchronously on Windows messages.

## External Dependencies
- `<windows.h>` — Win32 API types (`HWND`, `HINSTANCE`, `OSVERSIONINFO`, `LONG`, etc.)
- `<dinput.h>` — DirectInput 3.0 (input device enumeration/polling)
- `<dsound.h>` — DirectSound 3.0 (audio output)
- `<winsock.h>` / `<wsipx.h>` — Winsock + IPX networking
- `sysEventType_t`, `netadr_t`, `msg_t`, `usercmd_t`, `qboolean` — defined in `qcommon.h` / `q_shared.h` (engine shared headers, included transitively by including modules)
- `g_wv` — defined in `code/win32/win_main.c`
