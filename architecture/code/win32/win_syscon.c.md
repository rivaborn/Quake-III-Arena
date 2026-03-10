# code/win32/win_syscon.c

## File Purpose
Implements the Win32 system console window for Quake III Arena, providing a dedicated GUI console for the dedicated server and optional viewlog window for the client. It creates and manages a native Win32 popup window with a scrollable text buffer, command input line, and action buttons.

## Core Responsibilities
- Create and destroy the Win32 console popup window (`Sys_CreateConsole` / `Sys_DestroyConsole`)
- Handle Win32 window messages for the console and input-line subclassed control
- Append formatted output text to the scrollable edit buffer (`Conbuf_AppendText`)
- Poll and return text typed in the console input line (`Sys_ConsoleInput`)
- Show, hide, or minimize the console based on `visLevel` (`Sys_ShowConsole`)
- Display a flashing error banner when a fatal error occurs (`Sys_SetErrorText`)
- Relay quit/close commands from the console window back into the engine event queue

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WinConData` | struct | All state for the Win32 console window: HWNDs, fonts, brushes, text buffers, visibility level, and the saved input-line `WNDPROC` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_wcd` | `WinConData` | static (file) | Sole instance of the console window state; accessed by all functions in this file |
| `s_timePolarity` | `static qboolean` (inside `ConWndProc`) | static (local) | Toggled by a 1-second timer to produce the flashing error-text color effect |
| `s_totalChars` | `static unsigned long` (inside `Conbuf_AppendText`) | static (local) | Running count of characters appended; used to detect edit-control overflow and reset selection |

## Key Functions / Methods

### ConWndProc
- **Signature:** `static LONG WINAPI ConWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)`
- **Purpose:** Window procedure for the top-level console window; handles activation, close, color painting, button commands, creation, and the 1-second timer.
- **Inputs:** Standard Win32 `WNDPROC` parameters.
- **Outputs/Return:** `LONG` — 0 for handled messages, otherwise `DefWindowProc`.
- **Side effects:** On `WM_CREATE`, allocates two `HBRUSH` objects and starts a 1-second timer. On `WM_CLOSE` (dedicated), allocates a `"quit"` string via `CopyString` and posts it to the engine event queue. On `WM_COMMAND/QUIT_ID`, similarly posts quit.
- **Calls:** `SetFocus`, `Cvar_Set`, `CopyString`, `Sys_QueEvent`, `PostQuitMessage`, `Sys_ShowConsole`, `SendMessage`, `UpdateWindow`, `CreateSolidBrush`, `SetTimer`, `InvalidateRect`, `DefWindowProc`.
- **Notes:** The `WM_ERASEBKGND` logo-drawing path is entirely `#if 0`-disabled. `s_timePolarity` drives alternating text colors on the error box.

### InputLineWndProc
- **Signature:** `LONG WINAPI InputLineWndProc(HWND hWnd, UINT uMsg, WPARAM wParam, LPARAM lParam)`
- **Purpose:** Subclassed window procedure for the single-line input edit control; captures Enter key presses and appends the typed text to `s_wcd.consoleText`.
- **Inputs:** Standard `WNDPROC` parameters.
- **Outputs/Return:** 0 on Enter handled; otherwise delegates to original proc via `CallWindowProc`.
- **Side effects:** Writes into `s_wcd.consoleText` (read by `Sys_ConsoleInput`); calls `Sys_Print` to echo input; clears the input field via `SetWindowText`.
- **Calls:** `GetWindowText`, `strncat`, `strcat`, `SetWindowText`, `Sys_Print`, `va`, `CallWindowProc`.
- **Notes:** `WM_KILLFOCUS` guard prevents focus leaving to the main window or error box, keeping keyboard input in the console.

### Sys_CreateConsole
- **Signature:** `void Sys_CreateConsole(void)`
- **Purpose:** Registers the `"Q3 WinConsole"` window class and creates the full console window hierarchy: main window, scroll-buffer edit, input-line edit, Copy/Clear/Quit buttons, and fonts.
- **Inputs:** None (reads `g_wv.hInstance`).
- **Outputs/Return:** None.
- **Side effects:** Populates all HWND/HFONT/HBRUSH fields of `s_wcd`; subclasses the input-line via `SetWindowLong`; sets `s_wcd.visLevel = 1`; makes the window visible.
- **Calls:** `RegisterClass`, `AdjustWindowRect`, `GetDC`, `GetDeviceCaps`, `ReleaseDC`, `CreateWindowEx`, `CreateWindow`, `CreateFont`, `SendMessage`, `SetWindowLong`, `ShowWindow`, `UpdateWindow`, `SetForegroundWindow`, `SetFocus`.
- **Notes:** Window is centered on the desktop (540×450 client area). Returns silently on any creation failure.

### Sys_DestroyConsole
- **Signature:** `void Sys_DestroyConsole(void)`
- **Purpose:** Hides and destroys the console window, zeroing `s_wcd.hWnd`.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** `DestroyWindow` frees child windows and GDI objects owned by the window.
- **Calls:** `ShowWindow`, `CloseWindow`, `DestroyWindow`.

### Sys_ShowConsole
- **Signature:** `void Sys_ShowConsole(int visLevel, qboolean quitOnClose)`
- **Purpose:** Adjusts console visibility: 0 = hide, 1 = normal, 2 = minimize.
- **Inputs:** `visLevel` (0–2), `quitOnClose` flag stored in `s_wcd`.
- **Outputs/Return:** None.
- **Side effects:** Updates `s_wcd.visLevel` and `s_wcd.quitOnClose`; calls `ShowWindow`; on show-normal, scrolls buffer to end.
- **Calls:** `ShowWindow`, `SendMessage`, `Sys_Error`.

### Sys_ConsoleInput
- **Signature:** `char *Sys_ConsoleInput(void)`
- **Purpose:** Returns the most recently submitted console input line, or NULL if none pending.
- **Inputs:** None.
- **Outputs/Return:** Pointer to `s_wcd.returnedText` (static buffer), or NULL.
- **Side effects:** Clears `s_wcd.consoleText[0]` after copying.
- **Calls:** `strcpy`.

### Conbuf_AppendText
- **Signature:** `void Conbuf_AppendText(const char *pMsg)`
- **Purpose:** Converts a Q3 print string (stripping color codes, normalizing line endings) and appends it to the scroll-buffer edit control.
- **Inputs:** `pMsg` — null-terminated string, potentially containing `^X` color escapes.
- **Outputs/Return:** None.
- **Side effects:** Updates `s_totalChars`; sends `EM_REPLACESEL` to `s_wcd.hwndBuffer`; resets selection when overflow threshold (`0x7fff`) is exceeded.
- **Calls:** `strlen`, `Q_IsColorString`, `SendMessage`.
- **Notes:** Uses a local 32 KB intermediate buffer. Converts `\n` and `\r` to `\r\n` for Win32 edit controls.

### Sys_SetErrorText
- **Signature:** `void Sys_SetErrorText(const char *buf)`
- **Purpose:** Stores an error message and (on first call) creates a static error-box child window above the scroll buffer; destroys the input line.
- **Inputs:** `buf` — error string, copied into `s_wcd.errorString` (80-byte cap).
- **Outputs/Return:** None.
- **Side effects:** Creates `s_wcd.hwndErrorBox`; destroys `s_wcd.hwndInputLine`; sets window text.
- **Calls:** `Q_strncpyz`, `CreateWindow`, `SendMessage`, `SetWindowText`, `DestroyWindow`.

## Control Flow Notes
This file is **init/shutdown** scoped from the Win32 platform layer (`win_main.c`). `Sys_CreateConsole` is called early during engine startup. `Conbuf_AppendText` and `Sys_ConsoleInput` are called every frame from the common/server layer (`Com_Printf` routing and server input polling). `Sys_ShowConsole` is driven by `com_viewlog` cvar changes. `Sys_DestroyConsole` is called at shutdown.

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, cvar types, `Sys_Error`, `Sys_Print`, `Sys_QueEvent`, `CopyString`, `Cvar_Set`, `Q_IsColorString`, `Q_strncpyz`, `va`
- `win_local.h` — `WinVars_t g_wv` (hInstance), `Sys_QueEvent` declaration, Win32 headers (`windows.h`, DirectInput, DirectSound, WinSock)
- `resource.h` — `IDI_ICON1` resource ID
- `com_viewlog`, `com_dedicated` — cvars defined elsewhere in `qcommon`
- `SE_CONSOLE` — sysEventType enum value defined in `qcommon.h`
