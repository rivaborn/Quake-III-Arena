# code/win32/win_wndproc.c

## File Purpose
Implements the Win32 window procedure (`MainWndProc`) for Quake III Arena, translating Windows OS messages into engine input events and managing window lifecycle, focus, and Alt-Tab suppression.

## Core Responsibilities
- Translate Win32 keyboard/mouse messages into engine `SE_KEY`/`SE_CHAR` events via `Sys_QueEvent`
- Map Windows scan codes to Quake key numbers, disambiguating numpad vs. cursor keys
- Handle mouse wheel input for both legacy (MSH_MOUSEWHEEL) and modern (WM_MOUSEWHEEL) paths
- Manage application focus/activation state and mouse capture toggling
- Suppress Alt-Tab on NT vs. 9x using platform-specific Win32 APIs
- Handle window creation/destruction, position tracking, and quit on `WM_CLOSE`
- Toggle fullscreen mode on Alt+Enter (`WM_SYSKEYDOWN` + VK_Return)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WinVars_t` | struct (defined in `win_local.h`) | Global Win32 state: HWND, HINSTANCE, `activeApp`, `isMinimized`, `sysMsgTime` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `g_wv` | `WinVars_t` | global | Shared Win32 window/system state; defined here, extern elsewhere |
| `MSH_MOUSEWHEEL` | `static UINT` | static | Registered message ID for legacy Win95/NT3.51 mouse wheel |
| `s_alttab_disabled` | `static qboolean` | static | Tracks whether Alt-Tab suppression is currently active |
| `s_scantokey[128]` | `static byte[]` | static | Scan code → Quake keynum lookup table |
| `vid_xpos` | `cvar_t *` | file (module-level) | Archived window X position |
| `vid_ypos` | `cvar_t *` | file (module-level) | Archived window Y position |
| `r_fullscreen` | `cvar_t *` | file (module-level) | Fullscreen toggle cvar |

## Key Functions / Methods

### WIN_DisableAltTab
- **Signature:** `static void WIN_DisableAltTab(void)`
- **Purpose:** Prevents Alt-Tab from stealing focus while the game is running fullscreen
- **Inputs:** None (reads `s_alttab_disabled`, queries `arch` cvar)
- **Outputs/Return:** void
- **Side effects:** Registers a hotkey (WinNT) or sets `SPI_SCREENSAVERRUNNING` (Win9x); sets `s_alttab_disabled = qtrue`
- **Calls:** `Q_stricmp`, `Cvar_VariableString`, `RegisterHotKey`, `SystemParametersInfo`
- **Notes:** No-op if already disabled

### WIN_EnableAltTab
- **Signature:** `static void WIN_EnableAltTab(void)`
- **Purpose:** Restores Alt-Tab behavior when leaving fullscreen or destroying the window
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Unregisters hotkey or clears screensaver flag; sets `s_alttab_disabled = qfalse`
- **Calls:** `Q_stricmp`, `Cvar_VariableString`, `UnregisterHotKey`, `SystemParametersInfo`

### VID_AppActivate
- **Signature:** `static void VID_AppActivate(BOOL fActive, BOOL minimize)`
- **Purpose:** Updates engine focus state when the window gains/loses activation; toggles mouse capture
- **Inputs:** `fActive` — whether window is active; `minimize` — whether minimized
- **Outputs/Return:** void
- **Side effects:** Writes `g_wv.isMinimized`, `g_wv.activeApp`; calls `Key_ClearStates`, `IN_Activate`
- **Calls:** `Com_DPrintf`, `Key_ClearStates`, `IN_Activate`

### MapKey
- **Signature:** `static int MapKey(int key)`
- **Purpose:** Converts a Win32 `lParam` key value into a Quake keynum, differentiating extended (cursor) keys from numpad equivalents
- **Inputs:** `key` — raw Win32 `lParam` from `WM_KEYDOWN`/`WM_KEYUP`
- **Outputs/Return:** Quake keynum (`int`)
- **Side effects:** None
- **Calls:** None (table lookup via `s_scantokey`)
- **Notes:** Bit 24 of `lParam` is the "extended key" flag. Non-extended arrow/home/end/pgup/pgdn map to numpad variants.

### MainWndProc
- **Signature:** `LONG WINAPI MainWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)`
- **Purpose:** Central Win32 window message handler; dispatches all OS input and lifecycle events to the engine
- **Inputs:** Standard Win32 `WndProc` parameters
- **Outputs/Return:** `LONG`; 0 for handled messages, `DefWindowProc` otherwise
- **Side effects:** Queues engine input events via `Sys_QueEvent`; updates cvars (`vid_xpos`, `vid_ypos`, `r_fullscreen`); stores `g_wv.hWnd`; triggers `quit` command on `WM_CLOSE`
- **Calls:** `Sys_QueEvent`, `MapKey`, `IN_MouseEvent`, `IN_Activate`, `VID_AppActivate`, `SNDDMA_Activate`, `Cvar_Get`, `Cvar_SetValue`, `Cbuf_ExecuteText`, `Cbuf_AddText`, `WIN_DisableAltTab`, `WIN_EnableAltTab`, `AdjustWindowRect`, `RegisterWindowMessage`, `DefWindowProc`
- **Notes:** Logitech mouse wheel bug workaround uses a `flip` toggle to treat each WM_MOUSEWHEEL as alternating press/release. `WM_DISPLAYCHANGE` handling is `#if 0`'d out.

## Control Flow Notes
This file is purely **event-driven**; it has no per-frame update loop of its own. `MainWndProc` is registered as the window class procedure during `WM_CREATE` setup (elsewhere in `win_main.c`/`win_glimp.c`) and is called by the OS message pump (`PeekMessage`/`DispatchMessage`) in the main loop. On `WM_CREATE` it initializes cvars and registers the legacy mouse wheel message. On `WM_ACTIVATE` it feeds activation state into both the input and sound subsystems.

## External Dependencies
- `../client/client.h` — `cls`, `KEYCATCH_CONSOLE`, `Key_ClearStates`, `Cbuf_*`, `Cvar_*`, `Com_DPrintf`
- `win_local.h` — `WinVars_t`, `g_wv`, `IN_Activate`, `IN_MouseEvent`, `Sys_QueEvent`, `SNDDMA_Activate`
- `<windows.h>`, `<dinput.h>` — Win32 message constants, `HWND`, `BOOL`, `RegisterHotKey`, `SystemParametersInfo`
- **Defined elsewhere:** `in_mouse`, `in_logitechbug` (extern cvars from `win_input.c`); all `K_*` key constants (from `keys.h`); `Sys_QueEvent` (from `win_main.c`)
