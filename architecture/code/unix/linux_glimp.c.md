# code/unix/linux_glimp.c

## File Purpose
This file implements all Linux/X11-specific OpenGL display initialization, input handling, and SMP render-thread support for Quake III Arena. It provides the platform-specific `GLimp_*` and `IN_*` entry points that the renderer and client layers depend on. It manages the X11 display connection, GLX context, video mode switching, mouse/keyboard grabbing, and gamma control.

## Core Responsibilities
- Create and manage the X11 window and GLX rendering context (`GLW_SetMode`)
- Load the OpenGL shared library and initialize GL extensions (`GLW_LoadOpenGL`, `GLW_InitExtensions`)
- Handle X11 events: keyboard, mouse (relative/DGA), buttons, window changes (`HandleEvents`)
- Grab/ungrab mouse and keyboard for in-game input (`install_grabs`, `uninstall_grabs`)
- Set display gamma via XF86VidMode extension (`GLimp_SetGamma`, `GLW_InitGamma`)
- Swap front/back buffers each frame (`GLimp_EndFrame`)
- Optionally spawn a dedicated render thread using pthreads (`GLimp_SpawnRenderThread` and SMP helpers)
- Initialize and shut down the input subsystem (`IN_Init`, `IN_Shutdown`, `IN_Frame`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `rserr_t` | enum | Error codes returned by `GLW_SetMode` (OK, invalid fullscreen, invalid mode, unknown) |
| `glwstate_t` | struct (extern, defined in `unix_glw.h`) | Holds the `dlopen` handle to the OpenGL library and optional log file pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `glw_state` | `glwstate_t` | global | OpenGL library handle and log fp |
| `dpy` | `Display *` | static | X11 display connection |
| `scrnum` | `int` | static | X11 screen number |
| `win` | `Window` | static | X11 window handle |
| `ctx` | `GLXContext` | static | Active GLX rendering context |
| `mouse_avail` | `qboolean` | static | Whether mouse input is available |
| `mouse_active` | `qboolean` | static | Whether mouse grabs are currently installed |
| `mx`, `my` | `int` | static | Accumulated mouse delta |
| `mwx`, `mwy` | `int` | static | Last tracked pointer position (non-DGA) |
| `mouseResetTime` | `int` | static | Timestamp of last mouse warp, used to suppress settle events |
| `vidmode_ext` | `qboolean` | global | Whether XF86VidMode extension is available |
| `vidmodes` | `XF86VidModeModeInfo **` | static | Array of available video modes |
| `vidmode_active` | `qboolean` | static | Whether a video mode switch is active |
| `vidmode_InitialGamma` | `XF86VidModeGamma` | static | Saved gamma before game modifies it |
| `in_mouse`, `in_dgamouse`, `in_subframe`, `in_nograb` | `cvar_t *` | static/global | Input control cvars |
| `r_allowSoftwareGL`, `r_previousglDriver` | `cvar_t *` | global | Renderer cvars |
| `smpMutex` | `pthread_mutex_t` | static (SMP) | Mutex protecting SMP data handoff |
| `renderCommandsEvent`, `renderCompletedEvent` | `pthread_cond_t` | static (SMP) | Condition variables for render thread synchronization |
| `smpData` | `volatile void *` | static (SMP) | Pointer passed from front-end to render thread |

## Key Functions / Methods

### XLateKey
- **Signature:** `static char *XLateKey(XKeyEvent *ev, int *key)`
- **Purpose:** Translates an X11 key event into a Quake key code and a character buffer.
- **Inputs:** X key event pointer; output parameter `key` to receive Quake key constant.
- **Outputs/Return:** Pointer to static char buffer with UTF/ASCII text; sets `*key`.
- **Side effects:** None beyond the static buffer.
- **Calls:** `XLookupString`, `Q_stricmpn`, `ri.Printf`
- **Notes:** Falls back to raw char value from `XLookupString` buffer when no keysym match; handles ctrl-key range (1–26).

### install_grabs / uninstall_grabs
- **Signature:** `static void install_grabs(void)` / `static void uninstall_grabs(void)`
- **Purpose:** Grab (or release) the X pointer and keyboard for exclusive in-game input; sets up DGA direct mouse if requested.
- **Inputs:** None (uses module-level `dpy`, `win`, cvars).
- **Outputs/Return:** void
- **Side effects:** Changes X pointer acceleration; warps cursor to window center; sets `mouseResetTime`.
- **Calls:** `XWarpPointer`, `XGrabPointer`, `XGrabKeyboard`, `XGetPointerControl`, `XChangePointerControl`, `XF86DGADirectVideo`, `XF86DGAQueryVersion`, `ri.Cvar_Set`, `Sys_Milliseconds`

### HandleEvents
- **Signature:** `static void HandleEvents(void)`
- **Purpose:** Drains the X11 event queue each frame and converts events to engine `sysEvent_t` entries.
- **Inputs:** None.
- **Outputs/Return:** void
- **Side effects:** Calls `Sys_QueEvent` for key/mouse/char events; warps pointer back to center after non-DGA mouse motion.
- **Calls:** `XPending`, `XNextEvent`, `XLateKey`, `Sys_QueEvent`, `Sys_XTimeToSysTime`, `repeated_press`, `XWarpPointer`
- **Notes:** Mouse wheel buttons 4/5 map to `K_MWHEELUP`/`K_MWHEELDOWN`; X11 button 2/3 mapping is intentionally swapped for Q3.

### GLW_SetMode
- **Signature:** `int GLW_SetMode(const char *drivername, int mode, qboolean fullscreen)`
- **Purpose:** Opens the X display, selects a video mode, picks a GLX visual by iterating bit-depth fallbacks, creates the X window, and creates/activates the GLX context.
- **Inputs:** Driver name string, video mode index, fullscreen flag.
- **Outputs/Return:** `rserr_t` cast to `int` — `RSERR_OK` on success.
- **Side effects:** Opens `dpy`; sets `win`, `ctx`, `scrnum`; populates `glConfig.vidWidth/Height/colorBits/depthBits/stencilBits`; may switch video modes via `XF86VidModeSwitchToMode`.
- **Calls:** `XOpenDisplay`, `XF86VidModeQueryVersion`, `XF86VidModeGetAllModeLines`, `XF86DGAQueryVersion`, `qglXChooseVisual`, `XCreateWindow`, `XMapWindow`, `qglXCreateContext`, `qglXMakeCurrent`, `qglGetString`, `GLimp_Shutdown`, `R_GetModeInfo`, `ri.Printf`
- **Notes:** Iterates up to 16 combinations of color/depth/stencil bits before failing; rejects software Mesa unless `r_allowSoftwareGL` is set.

### GLimp_Init
- **Signature:** `void GLimp_Init(void)`
- **Purpose:** Top-level renderer initialization: registers cvars, installs X error handler, loads OpenGL library, queries GL strings, applies hardware-specific cvar defaults, and initializes extensions and gamma.
- **Inputs:** None.
- **Outputs/Return:** void
- **Side effects:** Calls `InitSig` twice; sets `glConfig.driverType/hardwareType`; may call `ri.Error` on failure.
- **Calls:** `GLW_LoadOpenGL`, `GLW_InitExtensions`, `GLW_InitGamma`, `XSetErrorHandler`, `ri.Cvar_Get`, `ri.Cvar_Set`, `Q_stristr`, `InitSig`

### GLimp_Shutdown
- **Signature:** `void GLimp_Shutdown(void)`
- **Purpose:** Tears down the GLX context, destroys the X window, restores gamma and video mode, closes the display.
- **Inputs:** None.
- **Outputs/Return:** void
- **Side effects:** Zeroes `glConfig` and `glState`; calls `QGL_Shutdown`; nulls `dpy`/`win`/`ctx`.
- **Calls:** `IN_DeactivateMouse`, `qglXDestroyContext`, `XDestroyWindow`, `XF86VidModeSwitchToMode`, `XF86VidModeSetGamma`, `XCloseDisplay`, `QGL_Shutdown`, `memset`

### GLimp_EndFrame
- **Signature:** `void GLimp_EndFrame(void)`
- **Purpose:** Swaps GLX buffers at end of frame and toggles GL logging.
- **Side effects:** Calls `qglXSwapBuffers`; calls `QGL_EnableLogging`.

### GLimp_SpawnRenderThread / GLimp_RendererSleep / GLimp_FrontEndSleep / GLimp_WakeRenderer (SMP block)
- **Purpose:** SMP support — spawn a dedicated render thread; front-end and render-thread synchronization via mutex + condition variables. Stubs provided when `SMP` is not defined.

### IN_Init / IN_Shutdown / IN_Frame
- **Purpose:** Register input cvars; set `mouse_avail`; call `IN_StartupJoystick`. `IN_Frame` deactivates mouse when console is open in windowed mode; otherwise activates it.

## Control Flow Notes
- **Init:** `GLimp_Init` → `GLW_LoadOpenGL` → `GLW_StartDriverAndSetMode` → `GLW_SetMode` (window + context creation) → `GLW_InitExtensions` → `GLW_InitGamma`.
- **Per-frame:** `Sys_SendKeyEvents` → `HandleEvents` drains X events; `IN_Frame` manages mouse activation; `GLimp_EndFrame` swaps buffers.
- **Shutdown:** `GLimp_Shutdown` restores gamma/video mode and closes display.
- **SMP path:** `GLimp_SpawnRenderThread` creates a pthread; `GLimp_WakeRenderer`/`GLimp_RendererSleep`/`GLimp_FrontEndSleep` hand off render command pointers under a mutex.

## External Dependencies
- **X11:** `<X11/Xlib.h>` (via GLX), `<X11/keysym.h>`, `<X11/cursorfont.h>`
- **XFree86 extensions:** `<X11/extensions/xf86dga.h>`, `<X11/extensions/xf86vmode.h>`
- **GLX:** `<GL/glx.h>`
- **pthreads:** `<pthread.h>`, `<semaphore.h>`
- **Dynamic linking:** `<dlfcn.h>` — `dlsym` used to resolve ARB extension function pointers from `glw_state.OpenGLLib`
- **Defined elsewhere:** `Sys_QueEvent`, `Sys_XTimeToSysTime`, `Sys_Milliseconds`, `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging`, `InitSig`, `IN_StartupJoystick`, `IN_JoyMove`, `glConfig`, `glState`, `ri` (refimport), `cls`, `com_developer`, all `r_*` cvars, all `q_*` string utilities.
