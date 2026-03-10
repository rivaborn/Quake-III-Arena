# code/win32/glw_win.h

## File Purpose
Declares the Win32-specific OpenGL window state structure (`glwstate_t`) and its global instance. It encapsulates all Win32/WGL handles and display configuration needed to manage the OpenGL rendering context on Windows.

## Core Responsibilities
- Define the `glwstate_t` struct holding all Win32 GL window state
- Store Win32 handles: device context (HDC), GL rendering context (HGLRC), OpenGL DLL instance
- Track desktop display properties (bit depth, resolution)
- Track fullscreen mode and pixel format initialization state
- Expose the global `glw_state` instance to other translation units
- Guard against inclusion on non-Win32 platforms via `#error`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `glwstate_t` | struct | Aggregates all Win32/WGL state required to create and manage an OpenGL rendering context |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `glw_state` | `glwstate_t` | global (extern) | Single instance of Win32 GL window state; defined in `win_glimp.c` |

## Key Functions / Methods

None. This is a pure header — no functions are declared or defined.

## Control Flow Notes

This header is consumed by the Win32 renderer/glimp layer (`win_glimp.c`, `win_qgl.c`). It is populated during renderer initialization (`GLimp_Init` / `WG_CreateWindow`) and torn down during `GLimp_Shutdown`. The `hDC`/`hGLRC` fields are live for the entire duration of the rendering subsystem. `cdsFullscreen` is set when `ChangeDisplaySettings` is called, and must be cleared on shutdown to restore the desktop mode. `log_fp` supports optional GL call logging.

## External Dependencies
- `<windows.h>` (implicit) — provides `WNDPROC`, `HDC`, `HGLRC`, `HINSTANCE`, `FILE`
- `qboolean` — defined in `q_shared.h` (engine-wide boolean typedef)
- `glwstate_t glw_state` — defined externally in `code/win32/win_glimp.c`
