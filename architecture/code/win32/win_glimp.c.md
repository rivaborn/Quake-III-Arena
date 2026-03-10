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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `rserr_t` | enum | Return codes from `GLW_SetMode`: OK, invalid fullscreen, invalid mode, unknown |
| `glwstate_t` | struct (extern, defined in `glw_win.h`) | Holds HDC, HGLRC, desktop geometry, fullscreen flag, log file pointer, pixel format set flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `glw_state` | `glwstate_t` | global | All Win32 GL window state (HDC, HGLRC, desktop info, etc.) |
| `s_classRegistered` | `static qboolean` | file-static | Guards one-time `RegisterClass` call |
| `r_allowSoftwareGL` | `cvar_t *` | global | Cvar: allow software-emulated pixel formats |
| `r_maskMinidriver` | `cvar_t *` | global | Cvar: treat any DLL as an ICD |
| `renderCommandsEvent` | `HANDLE` | global | SMP event: signals render thread to begin |
| `renderCompletedEvent` | `HANDLE` | global | SMP event: signals front end that render completed |
| `renderActiveEvent` | `HANDLE` | global | SMP event: signals main thread that render thread is active |
| `glimpRenderThread` | `void (*)(void)` | global | Pointer to the render thread's entry function |
| `renderThreadHandle` | `HANDLE` | global | Handle to the spawned render thread |
| `smpData` | `static void *` | file-static | Data pointer exchanged between front end and render thread |
| `wglErrors` | `static int` | file-static | Count of `wglMakeCurrent` failures in SMP path |

## Key Functions / Methods

### GLimp_Init
- **Signature:** `void GLimp_Init(void)`
- **Purpose:** Top-level renderer init entry point. Checks OS version, retrieves `hInstance`/`wndproc` from cvars, loads the OpenGL DLL, queries GL strings, applies per-chipset cvar overrides, and probes extensions.
- **Inputs:** None (reads many `r_*` cvars)
- **Outputs/Return:** void
- **Side effects:** Populates `glConfig` (vendor, renderer, version, extensions, hardware type, color/depth/stencil bits, fullscreen flag); sets per-chipset cvars (`r_picmip`, `r_textureMode`, etc.); calls `WG_CheckHardwareGamma`.
- **Calls:** `GLW_CheckOSVersion`, `GLW_StartOpenGL`, `qglGetString`, `GLW_InitExtensions`, `WG_CheckHardwareGamma`, `ri.Cvar_Get`, `ri.Cvar_Set`
- **Notes:** Must be called before any rendering. `hInstance` and `wndproc` are passed in via cvars set by `win_main.c`.

### GLimp_Shutdown
- **Signature:** `void GLimp_Shutdown(void)`
- **Purpose:** Tears down the entire GL subsystem: restores gamma, releases WGL context, releases DC, destroys the HWND, resets display settings, shuts down QGL, and zeroes `glConfig`/`glState`.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** All GL handles NULLed; `ChangeDisplaySettings(0,0)` if fullscreen; `fclose` on log file; `memset` on `glConfig` and `glState`.
- **Calls:** `WG_RestoreGamma`, `qwglMakeCurrent`, `qwglDeleteContext`, `ReleaseDC`, `DestroyWindow`, `ChangeDisplaySettings`, `QGL_Shutdown`

### GLimp_EndFrame
- **Signature:** `void GLimp_EndFrame(void)`
- **Purpose:** Called once per frame by the renderer to present the back buffer and manage swap interval.
- **Inputs:** None (reads `r_swapInterval`, `r_drawBuffer`, `glConfig.driverType`)
- **Outputs/Return:** void
- **Side effects:** May call `qwglSwapIntervalEXT`; calls `SwapBuffers`/`qwglSwapBuffers`; calls `QGL_EnableLogging`.
- **Notes:** Skips swap if `r_drawBuffer` is `GL_FRONT`.

### GLimp_LogComment
- **Signature:** `void GLimp_LogComment(char *comment)`
- **Purpose:** Writes a comment string to the optional GL log file.
- **Side effects:** `fprintf` to `glw_state.log_fp` if open.

### GLimp_SpawnRenderThread
- **Signature:** `qboolean GLimp_SpawnRenderThread(void (*function)(void))`
- **Purpose:** Creates three Win32 events and launches the render thread for SMP operation.
- **Calls:** `CreateEvent`, `CreateThread`

### GLimp_RendererSleep / GLimp_FrontEndSleep / GLimp_WakeRenderer
- **Purpose:** Three-part SMP handshake. `GLimp_WakeRenderer` posts work to the render thread; `GLimp_RendererSleep` waits for a command; `GLimp_FrontEndSleep` waits for render completion. Each transfers the WGL context between threads using `qwglMakeCurrent`.

### GLW_SetMode *(internal)*
- **Signature:** `static rserr_t GLW_SetMode(const char *drivername, int mode, int colorbits, qboolean cdsFullscreen)`
- **Purpose:** Resolves the target resolution via `R_GetModeInfo`, queries desktop capabilities, attempts `ChangeDisplaySettings` for fullscreen with fallback to next higher enumerated mode, then calls `GLW_CreateWindow`.
- **Side effects:** Modifies `glConfig.vidWidth/vidHeight/displayFrequency/isFullscreen`; may show a `MessageBox` for low desktop color depth.

### GLW_InitExtensions *(internal)*
- **Purpose:** Iterates known extensions (S3TC, `EXT_texture_env_add`, `WGL_EXT_swap_control`, `GL_ARB_multitexture`, `GL_EXT_compiled_vertex_array`, `WGL_3DFX_gamma_control`) and binds function pointers via `qwglGetProcAddress`.
- **Side effects:** Sets `glConfig.textureCompression`, `glConfig.textureEnvAddAvailable`, `glConfig.maxActiveTextures`; populates `qglMultiTexCoord2fARB`, `qglLockArraysEXT`, etc.

### Notes (minor helpers)
- `GLW_ChoosePFD` — manual replacement for `ChoosePixelFormat`; scores PFDs by stereo > color > depth > stencil priority.
- `GLW_CreatePFD` — zero-fills and populates a `PIXELFORMATDESCRIPTOR` from requested bits.
- `GLW_MakeContext` — sets pixel format and creates/makes-current the HGLRC; returns `TRY_PFD_*` status codes.
- `GLW_InitDriver` — obtains the HDC and makes two attempts at pixel format (with/without stencil).
- `GLW_CreateWindow` — registers the window class once, creates `HWND`, calls `GLW_InitDriver`.
- `GLW_LoadOpenGL` — classifies driver type (ICD/standalone/Voodoo), calls `QGL_Init`, then `GLW_StartDriverAndSetMode`.
- `GLW_StartOpenGL` — fallback chain: tries `r_glDriver`, then 3DFX driver, then `opengl32.dll`.
- `GLW_CheckOSVersion` — sets `glw_state.allowdisplaydepthchange` based on Win95 OSR2 / WinNT version.

## Control Flow Notes
- **Init:** `R_Init` → `GLimp_Init` → `GLW_StartOpenGL` → `GLW_LoadOpenGL` → `GLW_SetMode` → `GLW_CreateWindow` → `GLW_InitDriver` → `GLW_MakeContext`. Extensions and gamma are initialized at the tail of `GLimp_Init`.
- **Frame:** `RE_EndFrame` → `GLimp_EndFrame` (buffer swap).
- **Shutdown:** `RE_Shutdown` → `GLimp_Shutdown`.
- **SMP:** If `r_smp` is set, the renderer spawns a thread via `GLimp_SpawnRenderThread`; the three `GLimp_*Sleep/Wake` functions synchronize it each frame.

## External Dependencies
- `../renderer/tr_local.h` — `glConfig`, `glState`, `ri` (refimport), renderer cvars
- `../qcommon/qcommon.h` — `cvar_t`, `ri.Cvar_Get`, `ri.Error`, `ri.Printf`
- `glw_win.h` — `glwstate_t` definition
- `win_local.h` — `WinVars_t g_wv` (hWnd, hInstance), Win32 headers
- `resource.h` — `IDI_ICON1` icon resource
- **Defined elsewhere:** `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` (`win_qgl.c`); `WG_CheckHardwareGamma`, `WG_RestoreGamma` (`win_gamma.c`); `R_GetModeInfo` (renderer); all `qwgl*`/`qgl*` function pointers (QGL layer); `g_wv` (`win_main.c`).
