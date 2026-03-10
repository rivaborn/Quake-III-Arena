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

## External Dependencies
- `<windows.h>` (implicit) — provides `WNDPROC`, `HDC`, `HGLRC`, `HINSTANCE`, `FILE`
- `qboolean` — defined in `q_shared.h` (engine-wide boolean typedef)
- `glwstate_t glw_state` — defined externally in `code/win32/win_glimp.c`

# code/win32/resource.h
## File Purpose
Auto-generated Windows resource identifier header for the Quake III Arena Win32 build. It defines numeric IDs for embedded Win32 resources (icons, bitmaps, cursors, strings) referenced by `winquake.rc`.

## Core Responsibilities
- Define symbolic integer constants for Win32 resource IDs (icons, bitmaps, cursors, string tables)
- Provide APSTUDIO bookkeeping macros so Visual Studio's resource editor knows the next available ID values for each resource category
- Act as the bridge between the `.rc` resource script and C/C++ source code that references resources by name

## External Dependencies
- **Consumed by:** `code/win32/winquake.rc` (resource script referencing these IDs)
- **Potentially referenced by:** Win32 platform code in `code/win32/` that loads icons, cursors, or bitmaps via `LoadIcon`, `LoadCursor`, `LoadBitmap`, etc.
- No standard library includes; no external symbols are used or defined here.

| Resource Constant | Value | Kind |
|---|---|---|
| `IDS_STRING1` | 1 | String table entry |
| `IDI_ICON1` | 1 | Icon resource |
| `IDB_BITMAP1` | 1 | Bitmap resource |
| `IDB_BITMAP2` | 128 | Bitmap resource |
| `IDC_CURSOR1` | 129 | Cursor resource |
| `IDC_CURSOR2` | 130 | Cursor resource |
| `IDC_CURSOR3` | 131 | Cursor resource |

# code/win32/win_gamma.c
## File Purpose
Manages hardware gamma ramp correction on Win32, using either the 3Dfx-specific WGL extension or the standard Win32 `SetDeviceGammaRamp` API. It saves the original gamma on init, applies game-specified gamma tables per frame, and restores the original on shutdown.

## Core Responsibilities
- Detect whether the hardware/driver supports gamma ramp modification (`WG_CheckHardwareGamma`)
- Save the pre-game hardware gamma ramp for later restoration
- Validate saved gamma ramp sanity (monotonically increasing, crash-recovery linear fallback)
- Apply per-channel RGB gamma ramp tables to the display device (`GLimp_SetGamma`)
- Apply Windows 2000-specific gamma clamping restrictions
- Enforce monotonically increasing gamma values before submission
- Restore original hardware gamma on game exit (`WG_RestoreGamma`)

## External Dependencies
- `<assert.h>` — standard C (unused in active code paths)
- `../renderer/tr_local.h` — `glConfig` (`glconfig_t`), `ri` (`refimport_t`), `r_ignorehwgamma` cvar
- `../qcommon/qcommon.h` — `Com_DPrintf`, `Com_Printf`
- `glw_win.h` — `glw_state` (`glwstate_t`), exposes `glw_state.hDC`
- `win_local.h` — Win32 headers (`windows.h`), `OSVERSIONINFO`, `GetVersionEx`
- `qwglSetDeviceGammaRamp3DFX`, `qwglGetDeviceGammaRamp3DFX` — defined elsewhere (WGL extension pointers, loaded in `win_glimp.c`)
- `glConfig.deviceSupportsGamma`, `glConfig.driverType` — defined in renderer globals (`tr_init.c`)

# code/win32/win_glimp.c
## File Purpose
Win32-specific OpenGL initialization, window management, and frame presentation layer for Quake III Arena. It implements the platform-facing `GLimp_*` interface required by the renderer, handling everything from pixel format selection and WGL context creation to fullscreen CDS mode switching and optional SMP render thread synchronization.

## Core Responsibilities
- Create and destroy the Win32 application window (`HWND`)
- Select an appropriate `PIXELFORMATDESCRIPTOR` and establish a WGL rendering context
- Handle fullscreen mode switching via `ChangeDisplaySettings` (CDS)
- Load an OpenGL DLL (ICD, standalone, or Voodoo) and bind all function pointers via `QGL_Init`
- Probe and enable supported OpenGL/WGL extensions (multitexture, S3TC, swap control, 3DFX gamma, CVA)
- Perform per-frame buffer swap and swap-interval management in `GLimp_EndFrame`
- Provide SMP support: spawn a render thread and coordinate it with event objects

## External Dependencies
- `../renderer/tr_local.h` — `glConfig`, `glState`, `ri` (refimport), renderer cvars
- `../qcommon/qcommon.h` — `cvar_t`, `ri.Cvar_Get`, `ri.Error`, `ri.Printf`
- `glw_win.h` — `glwstate_t` definition
- `win_local.h` — `WinVars_t g_wv` (hWnd, hInstance), Win32 headers
- `resource.h` — `IDI_ICON1` icon resource
- **Defined elsewhere:** `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` (`win_qgl.c`); `WG_CheckHardwareGamma`, `WG_RestoreGamma` (`win_gamma.c`); `R_GetModeInfo` (renderer); all `qwgl*`/`qgl*` function pointers (QGL layer); `g_wv` (`win_main.c`).

# code/win32/win_input.c
## File Purpose
Win32-specific input handling for Quake III Arena, managing mouse (both Win32 raw and DirectInput), joystick, and MIDI controller input. It translates hardware input events into engine-queued system events via `Sys_QueEvent`.

## Core Responsibilities
- Initialize, activate, deactivate, and shut down Win32 mouse and DirectInput mouse
- Poll DirectInput buffered mouse data and queue button/wheel/motion events
- Initialize and poll Win32 Multimedia joystick API, mapping axes and buttons to key events
- Initialize and receive MIDI input, mapping MIDI notes to aux key events
- Per-frame input dispatch (`IN_Frame`), including delayed DirectInput init fallback
- Register input-related cvars (`in_mouse`, `in_joystick`, `in_midi`, etc.)

## External Dependencies
- `../client/client.h` — `cls` (keyCatchers), `Cvar_*`, `Cmd_*`, `Com_Printf`, `Sys_QueEvent`, key constants
- `win_local.h` — `g_wv` (hWnd, hInstance, osversion, sysMsgTime), Win32/DInput/DSound headers
- Win32 APIs: `dinput.dll` (loaded dynamically), `winmm` (joystick/MIDI via `joyGetPosEx`, `midiInOpen`)
- Defined elsewhere: `Sys_QueEvent`, `Cvar_Set/Get/VariableValue`, `Com_Printf`, `Com_Memset`, `g_wv`, `cls`

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

## External Dependencies
- `<windows.h>` — Win32 API types (`HWND`, `HINSTANCE`, `OSVERSIONINFO`, `LONG`, etc.)
- `<dinput.h>` — DirectInput 3.0 (input device enumeration/polling)
- `<dsound.h>` — DirectSound 3.0 (audio output)
- `<winsock.h>` / `<wsipx.h>` — Winsock + IPX networking
- `sysEventType_t`, `netadr_t`, `msg_t`, `usercmd_t`, `qboolean` — defined in `qcommon.h` / `q_shared.h` (engine shared headers, included transitively by including modules)
- `g_wv` — defined in `code/win32/win_main.c`

# code/win32/win_main.c
## File Purpose
Win32-specific platform entry point and system abstraction layer for Quake III Arena. It implements `WinMain`, the OS event loop, file system enumeration, DLL loading, and miscellaneous system services required by the engine's `Sys_*` API contract.

## Core Responsibilities
- Owns `WinMain` and the top-level game loop (`IN_Frame` → `Com_Frame`)
- Queues and dispatches system events (keyboard, mouse, network, console) via `Sys_QueEvent` / `Sys_GetEvent`
- Provides Win32 filesystem services: directory listing, filtered file enumeration, `Sys_Mkdir`, `Sys_Cwd`
- Loads and unloads game module DLLs (`Sys_LoadDll` / `Sys_UnloadDll`) with a security warning in release builds
- Implements stub background file streaming (`Sys_InitStreamThread` etc.) — full threaded implementation is `#if 0`'d out
- Initializes the system: high-resolution timer, OS version detection, CPU identification, input subsystem
- Handles fatal error and clean-quit paths (`Sys_Error`, `Sys_Quit`)

## External Dependencies

- `../client/client.h` — `IN_Frame`, `IN_Init`, `IN_Shutdown`
- `../qcommon/qcommon.h` — `Com_Init`, `Com_Frame`, `NET_Init`, `NET_Restart`, `Sys_Milliseconds`, `Z_Malloc`, `Z_Free`, `FS_Read`, `FS_Seek`, `Cvar_*`, `Cmd_AddCommand`, `MSG_Init`
- `win_local.h` — `WinVars_t g_wv`, `Sys_CreateConsole`, `Sys_DestroyConsole`, `Sys_ConsoleInput`, `Sys_GetPacket`, `MainWndProc`, `Conbuf_AppendText`, `Sys_ShowConsole`, `Sys_SetErrorText`
- Win32 API: `<windows.h>`, `timeBeginPeriod`/`timeEndPeriod`/`timeGetTime` (winmm), `LoadLibrary`, `GetProcAddress`, `FreeLibrary`, `GlobalMemoryStatus`, `_findfirst`/`_findnext`/`_findclose`
- **Defined elsewhere:** `FS_BuildOSPath`, `Sys_GetProcessorId`, `Sys_GetCurrentUser`, `Sys_Milliseconds`, `CopyString`, `Q_strncpyz`, `Com_sprintf`, `Com_FilterPath`

# code/win32/win_net.c
## File Purpose
Windows-specific (Winsock) implementation of the low-level network layer for Quake III Arena. It creates and manages UDP sockets for IP and IPX protocols, handles SOCKS5 proxy tunneling, and provides packet send/receive primitives consumed by the platform-independent `qcommon` network layer.

## Core Responsibilities
- Initialize and shut down the Winsock library (`WSAStartup`/`WSACleanup`)
- Open, configure, and close UDP sockets for IP (`ip_socket`) and IPX (`ipx_socket`) protocols
- Implement optional SOCKS5 proxy negotiation and UDP-associate relay
- Convert between engine `netadr_t` and OS `sockaddr`/`sockaddr_ipx` representations
- Receive incoming packets (`Sys_GetPacket`) and send outgoing packets (`Sys_SendPacket`)
- Classify remote addresses as LAN or WAN (`Sys_IsLANAddress`)
- Enumerate and cache local IP addresses for LAN detection

## External Dependencies
- `<winsock.h>`, `<wsipx.h>` — Winsock and IPX socket APIs (via `win_local.h`)
- `../game/q_shared.h` — `qboolean`, `byte`, `cvar_t`, `netadr_t` type definitions
- `../qcommon/qcommon.h` — `msg_t`, `NET_AdrToString`, `Com_Printf`, `Com_Error`, `Cvar_Get`, `Cvar_SetValue`, `PORT_ANY`, `PORT_SERVER`, `NA_*` address type constants
- `NET_AdrToString` — defined in `qcommon/net_chan.c`, not in this file
- `NET_SendPacket` (higher-level wrapper) — defined in `qcommon/net_chan.c`

# code/win32/win_qgl.c
## File Purpose
Windows-specific binding layer that dynamically loads `opengl32.dll` (or a 3Dfx Glide wrapper) and assigns all OpenGL 1.x and WGL function pointers to the engine's `qgl*`/`qwgl*` indirection layer. It also implements an optional per-call logging path that intercepts every GL call and writes a human-readable trace to a log file.

## Core Responsibilities
- Load an OpenGL DLL via `LoadLibrary` and resolve all `gl*`/`wgl*` symbols via `GetProcAddress` (`QGL_Init`)
- Null-out and free the DLL handle on shutdown (`QGL_Shutdown`)
- Maintain two parallel function-pointer sets: `dll*` (direct DLL pointers) and `qgl*`/`qwgl*` (active pointers used by the renderer)
- Swap active pointers between direct (`dll*`) and logging (`log*`) wrappers on demand (`QGL_EnableLogging`)
- Emit per-call human-readable GL traces to a timestamped `gl.log` file when logging is enabled
- Validate 3Dfx Glide availability before loading the 3Dfx driver

## External Dependencies
- `#include <float.h>` — standard C
- `#include "../renderer/tr_local.h"` — provides `ri` (refimport), `r_logFile` cvar, `glconfig_t`, renderer types
- `#include "glw_win.h"` — provides `glwstate_t` and `glw_state` (the Win32 GL window/context state)
- **Defined elsewhere:** `glw_state` (defined in `win_glimp.c`); `ri` (renderer import table); `r_logFile`, `qglActiveTextureARB`, `qglClientActiveTextureARB`, `qglMultiTexCoord2fARB`, `qglLockArraysEXT`, `qglUnlockArraysEXT` (declared/used in renderer modules); Windows API: `LoadLibrary`, `FreeLibrary`, `GetProcAddress`, `GetSystemDirectory`

# code/win32/win_shared.c
## File Purpose
Provides Windows-specific implementations of shared system services required by the Quake III engine, including timing, floating-point snapping, CPU feature detection, and user/path queries. This file bridges the platform-agnostic `Sys_*` interface declared in `qcommon.h` to Win32 APIs.

## Core Responsibilities
- Provide `Sys_Milliseconds` using `timeGetTime()` with a stable epoch base
- Implement `Sys_SnapVector` via x86 FPU inline assembly (`fistp`) for fast float-to-int truncation
- Detect CPU capabilities (Pentium, MMX, 3DNow!, KNI/SSE) via CPUID and return a capability constant
- Query the Windows username via `GetUserName`
- Provide default home/install path resolution

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `qtrue`/`qfalse`, shared types
- `../qcommon/qcommon.h` — `CPUID_*` constants, `Sys_*` declarations, `Sys_Cwd`
- `win_local.h` — `WinVars_t`, Win32 subsystem headers
- `<windows.h>` (via `win_local.h`) — `GetUserName`
- `<mmsystem.h>` (implicit via WinMM link) — `timeGetTime`
- `Sys_Cwd` — defined elsewhere (not in this file)

# code/win32/win_snd.c
## File Purpose
Windows-specific DirectSound DMA backend for Quake III Arena's audio system. It implements the platform sound device interface (`SNDDMA_*`) using DirectSound COM APIs to drive a looping secondary buffer that the portable mixer writes into.

## Core Responsibilities
- Initialize and tear down a DirectSound device via COM (`CoCreateInstance`)
- Create and configure a secondary DirectSound buffer (hardware-preferred, software fallback)
- Lock/unlock the circular DMA buffer each frame so the mixer can write samples
- Report the current playback position within the DMA ring buffer
- Re-establish the cooperative level when the application window changes focus

## External Dependencies
- `../client/snd_local.h` — `dma_t dma`, `channel_t`, `SNDDMA_*` declarations, `S_Shutdown`
- `win_local.h` — `WinVars_t g_wv` (for `hWnd`), DirectSound/DirectInput version defines, Win32 headers
- `<dsound.h>`, `<windows.h>` — DirectSound COM interfaces
- `Com_Printf`, `Com_DPrintf` — defined in `qcommon`
- `g_wv.hWnd` — window handle from the Win32 platform layer
- `S_Shutdown` — portable sound shutdown, defined in `client/snd_dma.c`

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

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, cvar types, `Sys_Error`, `Sys_Print`, `Sys_QueEvent`, `CopyString`, `Cvar_Set`, `Q_IsColorString`, `Q_strncpyz`, `va`
- `win_local.h` — `WinVars_t g_wv` (hInstance), `Sys_QueEvent` declaration, Win32 headers (`windows.h`, DirectInput, DirectSound, WinSock)
- `resource.h` — `IDI_ICON1` resource ID
- `com_viewlog`, `com_dedicated` — cvars defined elsewhere in `qcommon`
- `SE_CONSOLE` — sysEventType enum value defined in `qcommon.h`

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

## External Dependencies
- `../client/client.h` — `cls`, `KEYCATCH_CONSOLE`, `Key_ClearStates`, `Cbuf_*`, `Cvar_*`, `Com_DPrintf`
- `win_local.h` — `WinVars_t`, `g_wv`, `IN_Activate`, `IN_MouseEvent`, `Sys_QueEvent`, `SNDDMA_Activate`
- `<windows.h>`, `<dinput.h>` — Win32 message constants, `HWND`, `BOOL`, `RegisterHotKey`, `SystemParametersInfo`
- **Defined elsewhere:** `in_mouse`, `in_logitechbug` (extern cvars from `win_input.c`); all `K_*` key constants (from `keys.h`); `Sys_QueEvent` (from `win_main.c`)

