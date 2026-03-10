# Subsystem Overview

## Purpose
The `code/win32` subsystem is the Win32 platform abstraction layer for Quake III Arena, providing all OS-specific implementations required by the engine's `Sys_*`, `GLimp_*`, `IN_*`, and `SNDDMA_*` interface contracts. It bridges the portable engine core (`qcommon`, renderer, client, server) to Windows APIs covering windowing, OpenGL context management, input devices, audio, networking, and system services.

## Key Files

| File | Role |
|---|---|
| `win_main.c` | `WinMain` entry point, game loop, OS event dispatch, DLL loading, filesystem services |
| `win_glimp.c` | OpenGL window creation, WGL context setup, fullscreen switching, SMP render thread, `GLimp_*` interface |
| `win_qgl.c` | Dynamic OpenGL DLL loader; binds all `qgl*`/`qwgl*` function pointers; optional per-call GL trace logging |
| `win_gamma.c` | Hardware gamma ramp save/apply/restore via `SetDeviceGammaRamp` or 3Dfx WGL extension |
| `win_input.c` | Mouse (Win32 raw + DirectInput), joystick (WinMM), and MIDI input; queues `SE_KEY`/motion events |
| `win_wndproc.c` | `MainWndProc`: translates Win32 messages to engine key/char/focus events; Alt-Tab and fullscreen handling |
| `win_snd.c` | DirectSound DMA backend; drives looping secondary buffer for the portable audio mixer (`SNDDMA_*`) |
| `win_net.c` | Winsock UDP sockets for IP and IPX; SOCKS5 proxy; `Sys_GetPacket`/`Sys_SendPacket` |
| `win_shared.c` | `Sys_Milliseconds`, `Sys_SnapVector` (x86 FPU), CPU detection (CPUID/MMX/SSE), username/path queries |
| `win_syscon.c` | Win32 GUI system console window (dedicated server viewlog, error banner, command input) |
| `glw_win.h` | Declares `glwstate_t` (HDC, HGLRC, fullscreen state) and `glw_state` global |
| `win_local.h` | Declares `WinVars_t` (`g_wv`), `IN_*`/`SNDDMA_*`/console API, `MainWndProc`; includes all Win32 SDK headers |
| `resource.h` | Win32 resource ID constants (icons, bitmaps, cursors) consumed by `winquake.rc` and platform code |

## Core Responsibilities

- **Window and OpenGL context lifecycle:** Create the `HWND`, select a `PIXELFORMATDESCRIPTOR`, establish a WGL rendering context, and tear it all down cleanly via the `GLimp_*` interface.
- **OpenGL function pointer binding:** Dynamically load `opengl32.dll` (or a 3Dfx wrapper) at runtime and resolve every `qgl*`/`qwgl*` symbol; optionally redirect all calls through per-call logging wrappers.
- **Display mode management:** Switch between windowed and fullscreen via `ChangeDisplaySettings` (CDS); save and restore hardware gamma ramps on init and shutdown.
- **Input device management:** Initialize, poll, and shut down Win32 raw mouse, DirectInput mouse, WinMM joystick, and MIDI; translate hardware events into engine `Sys_QueEvent` calls.
- **Audio DMA backend:** Initialize a DirectSound looping secondary buffer and provide lock/unlock/position access so the portable mixer can write PCM samples each frame.
- **Network socket layer:** Initialize Winsock, create and manage UDP sockets for IP and IPX, implement optional SOCKS5 relay, and provide `Sys_GetPacket`/`Sys_SendPacket` to `qcommon`.
- **System services:** Provide `WinMain`, the top-level game loop, high-resolution timing, CPU capability detection, user/path queries, DLL loading (`Sys_LoadDll`), and fatal error/quit handling.
- **System console:** Display a native Win32 popup console for dedicated-server output, command input, and fatal error banners.

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `GLimp_Init`, `GLimp_Shutdown`, `GLimp_EndFrame`, `GLimp_SetGamma` — consumed by the renderer (`tr_init.c`, `tr_backend.c`)
- `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` — consumed by renderer startup/shutdown
- `IN_Init`, `IN_Shutdown`, `IN_Frame` — consumed by `client/cl_input.c` and `win_main.c` game loop
- `SNDDMA_Init`, `SNDDMA_BeginPainting`, `SNDDMA_Submit`, `SNDDMA_GetDMAPos`, `SNDDMA_Activate` — consumed by `client/snd_dma.c`
- `Sys_GetPacket`, `Sys_SendPacket`, `Sys_IsLANAddress` — consumed by `qcommon/net_chan.c`
- `Sys_Milliseconds`, `Sys_SnapVector`, `Sys_GetProcessorId`, `Sys_GetCurrentUser` — consumed by `qcommon` and game modules
- `Sys_LoadDll`, `Sys_UnloadDll` — consumed by `qcommon` VM/module loader
- `Sys_CreateConsole`, `Sys_DestroyConsole`, `Conbuf_AppendText`, `Sys_ShowConsole` — consumed by `qcommon/common.c`
- `MainWndProc` — registered as the Win32 window class procedure by `win_glimp.c`
- `glw_state` (`glwstate_t`) — shared across `win_glimp.c`, `win_qgl.c`, `win_gamma.c`
- `g_wv` (`WinVars_t`) — shared across all `win32/` modules and `win_snd.c`

**Consumed from other subsystems:**
- `renderer/tr_local.h`: `glConfig`, `glState`, `ri` (refimport table), renderer cvars (`r_ignorehwgamma`, `r_logFile`, etc.)
- `qcommon/qcommon.h`: `Com_Init`, `Com_Frame`, `Com_Printf`, `Com_Error`, `Cvar_*`, `Cmd_*`, `NET_*`, `Sys_QueEvent` declarations, `sysEventType_t`
- `client/client.h`: `cls`, key catcher constants, `Key_ClearStates`, `Cbuf_*`
- `client/snd_local.h`: `dma_t dma`, `S_Shutdown`
- `game/q_shared.h`: `qboolean`, `netadr_t`, `byte`, engine-wide shared types
- `resource.h`: `IDI_ICON1` for window icon registration

## Runtime Role

**Init:**
- `WinMain` (`win_main.c`) calls `Com_Init`, initializes the high-resolution timer (`timeBeginPeriod`), detects OS version, and enters the game loop.
- `GLimp_Init` (`win_glimp.c`) registers the window class (using `IDI_ICON1`), creates `HWND`, runs pixel format selection, creates the WGL context, loads the OpenGL DLL via `QGL_Init`, and probes WGL extensions.
- `WG_CheckHardwareGamma` (`win_gamma.c`) saves the current desktop gamma ramp.
- `IN_Init` (`win_input.c`) registers input cvars and optionally initializes DirectInput.
- `SNDDMA_Init` (`win_snd.c`) creates the DirectSound device and allocates the DMA ring buffer.
- `NET_Init` / Winsock startup (`win_net.c`) opens UDP sockets.
- `Sys_CreateConsole` (`win_syscon.c`) creates the GUI system console window.

**Frame:**
- `win_main.c` game loop: dispatches Win32 messages (→ `MainWndProc` → `Sys_QueEvent`), calls `IN_Frame` (input polling), then `Com_Frame` (full engine tick).
- `GLimp_EndFrame` (`win_glimp.c`) calls `qwglSwapBuffers` (with optional swap-interval control).
- `SNDDMA_BeginPainting` / `SNDDMA_Submit` (`win_snd.c`) lock and unlock the DirectSound buffer around the portable mixer write.
- `Sys_GetPacket` (`win_net.c`) drains incoming UDP datagrams into the engine event queue.

**Shutdown:**
- `WG_RestoreGamma` (`win_gamma.c`) restores the saved desktop gamma ramp.
- `QGL_Shutdown` (`win_qgl.c`) nulls all `qgl*` pointers and frees the OpenGL DLL handle.
- `GLimp_Shutdown` (`win_glimp.c`) destroys the WGL context, the `HWND`, and reverts any CDS fullscreen mode.
- `SNDDMA_Shutdown` (`win_snd.c`) releases DirectSound COM objects.
- Winsock `WSACleanup` (`win_net.c`) closes sockets and unloads Winsock.
- `Sys_DestroyConsole` (`win_syscon.c`) destroys the console popup window.

## Notable Implementation Details

- **QGL indirection layer:** All OpenGL calls go through `qgl*` function pointers rather than direct `gl*` symbols. `win_qgl.c` maintains two parallel pointer sets (`dll*` direct and `log*` logging wrappers) and swaps the active `qgl*` set at runtime, enabling zero-overhead GL call tracing without recompilation.
- **Gamma ramp fallback:** `win_gamma.c` validates the saved gamma ramp for monotonicity at startup and substitutes a linear ramp if the saved data is corrupt — a crash-recovery path for drivers that report nonsensical gamma tables.
- **Dual mouse backends:** `win_input.c` supports both Win32 raw mouse (`SetCursorPos`/`GetCursorPos`) and DirectInput buffered mouse, with a deferred DirectInput initialization fallback invoked from `IN_Frame` to handle timing edge cases.
- **SMP render thread:** `win_glimp.c` contains a symmetric multiprocessing render thread path guarded by `#ifdef SMP`, using Win32 event objects to synchronize the game thread and a dedicated OpenGL submission thread.
- **SOCKS5 proxy:** `win_net.c` implements a full SOCKS5 UDP-associate handshake for routing game traffic through a proxy, controlled by `net_socksServer` / `net_socksPort` cvars.
- **Alt-Tab suppression:** `win_wndproc.c` uses platform-version-conditional code — `RegisterHotKey` on NT-family systems and `SystemParametersInfo(SPI_SETSCREENSAVERRUNNING)` on Win9x — to suppress system key interception during fullscreen play.
- **Threaded file streaming stub:** `win_main.c` contains a complete background file streaming thread implementation that is compiled out (`#if 0`), with only stub no-op functions active in the shipped code.
