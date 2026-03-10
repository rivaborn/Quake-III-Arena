# code/unix/linux_glimp.c — Enhanced Analysis

## Architectural Role

This file is the **platform-specific GL/input bridge** connecting three major subsystems:

1. **Renderer** (`code/renderer/tr_*.c`) — depends on `GLimp_*` for window creation, context management, buffer swapping, and optional SMP thread coordination
2. **Client** (`code/client/cl_*.c`) — depends on `IN_*` for keyboard/mouse event processing and joystick input
3. **qcommon** (`code/qcommon/`) — provides foundational services via refimport (`ri.*` function pointers) and platform calls (`Sys_*`)

Unlike the renderer's stateless draw-call abstraction, this file is deeply stateful—holding the X11 display connection, GLX context, and input device state across the entire game session.

## Key Cross-References

### Incoming (who calls this file's functions)
- **`code/renderer/tr_init.c`** → `GLimp_Init`, `GLimp_Shutdown` (renderer module lifecycle)
- **`code/renderer/tr_backend.c`** → `GLimp_EndFrame` (per-frame buffer swap); `GLimp_SpawnRenderThread`, `GLimp_WakeRenderer`, `GLimp_FrontEndSleep`, `GLimp_RendererSleep` (SMP synchronization)
- **`code/renderer/tr_main.c`** → `GLimp_SetGamma` (gamma correction)
- **`code/client/cl_main.c`** → `IN_Init`, `IN_Shutdown`, `IN_Frame` (input subsystem lifecycle)
- **`code/client/cl_input.c`** → `Sys_SendKeyEvents` (keyboard event draining each frame)
- **`code/qcommon/common.c`** → indirectly via `Com_Frame` event loop

### Outgoing (what this file depends on)
- **`code/renderer/tr_local.h`** / `qcommon/qcommon.h` / `client/client.h` — type definitions and globals
- **`code/qcommon/qcommon.h`** → `ri.Printf`, `ri.Error`, `ri.Cvar_*`, `ri.Hunk_Alloc` (refimport vtable)
- **`code/client/client.h`** → `Sys_QueEvent` (event queue enqueue); `cls` global (console/UI state)
- **`code/unix/linux_main.c`** → `Sys_Milliseconds`, `Sys_XTimeToSysTime` (time conversion)
- **`code/unix/linux_local.h`** → `IN_JoyMove` (joystick input from `linux_joystick.c`)
- **Dynamic GL function resolution** → `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` (via dlsym'd function pointers in `glw_state.OpenGLLib`)

## Design Patterns & Rationale

### SMP Render Thread Synchronization
The `GLimp_SpawnRenderThread` / `GLimp_WakeRenderer` / `GLimp_FrontEndSleep` / `GLimp_RendererSleep` block implements a **producer-consumer mutex + condition-variable pattern** for optional front-end/back-end separation:
- Front-end (scene traversal, sorting) runs on main thread
- Back-end (GL command execution) runs on dedicated render thread
- Handoff via `smpData` pointer protected by `smpMutex` and signaled via `renderCommandsEvent` / `renderCompletedEvent`
- **Rationale:** Late 2000s dual-core commonality; separates I/O-bound geometry processing from GPU submission, though modern drivers have rendered this pattern less effective.

### X11 Event → Engine Event Translation
The `HandleEvents` + `XLateKey` pair converts low-level X11 keysyms and button events into engine-canonical `sysEvent_t` entries queued via `Sys_QueEvent`. 
- **Why:** X11 keyboard layout abstraction (supports French AZERTY, etc.); mouse button remapping intentionally swaps X11 buttons 2/3 (middle/right) for Q3 conventions
- **Keyboard handling note:** Falls back to raw char value from `XLookupString` when keysym fails, handling ctrl-key ranges (1–26) correctly—avoids the "qwerty on French keyboard" problem mentioned in comments

### Dual Mouse Control Paths
`install_grabs` branches on `in_dgamouse->value`:
- **DGA path:** `XF86DGADirectVideo(dpy, DefaultScreen(dpy), XF86DGADirectMouse)` for raw hardware acceleration (eliminates cursor clipping, warping overhead)
- **Standard path:** Manual position tracking via `mwx`/`mwy` with pointer acceleration restoration on release
- **Rationale:** Hardware DGA was faster on older X11; fallback ensures compatibility with systems lacking XFree86 DGA extension

### Fallback Visual Selection Iteration
`GLW_SetMode` iterates 16 combinations of `(colorBits, depthBits, stencilBits)` via `qglXChooseVisual`, accepting the first match. **Rationale:** GLX visuals are pre-computed by X server; no on-demand creation possible. Tries 32-bit color first, cascades to 16-bit rather than failing.

## Data Flow Through This File

**Initialization path:**
```
GLimp_Init
  → GLW_LoadOpenGL (dlopen the GL library)
  → GLW_StartDriverAndSetMode (iterates modes)
    → GLW_SetMode (creates X window + GLX context)
  → GLW_InitExtensions (resolve ARB function pointers)
  → GLW_InitGamma (save initial gamma for restore-on-exit)
```

**Per-frame input path:**
```
Sys_SendKeyEvents (called from client frame loop)
  → HandleEvents (drains XNextEvent queue)
    → XLateKey (X keysym → Quake key code)
    → Sys_QueEvent (enqueue sysEvent_t)
  → IN_Frame (mouse activation/deactivation based on UI state)
    → install_grabs / uninstall_grabs (X pointer/keyboard control)
```

**SMP render thread path (if enabled):**
```
Renderer front-end (main thread)
  → GLimp_FrontEndSleep (wait for back-end to finish)
  ← Renderer back-end (pthread)
    ← GLimp_WakeRenderer (signal new commands available)
```

## Learning Notes

1. **X11 era idioms (2005):** Direct gamma control via XF86VidMode, DGA mouse acceleration suppression, and video mode switching through extension queries—all artifacts of X11 server hardware abstraction before compositing managers.

2. **pthread SMP pattern:** The condition-variable handoff is textbook producer-consumer, but modern engines (post-2010) typically use lockless ring buffers or dispatch queues for render command decoupling.

3. **Keyboard abstraction maturity:** Unlike simple scan-code mapping, the fallback to `XLookupString` character buffers shows awareness that key meaning is locale-dependent—a lesson modern Wayland/web platforms revisit.

4. **Mouse input fragmentation:** The dual DGA/relative path reflects early-2000s Linux gaming fragmentation; modern Linux games typically use libinput directly, bypassing X11 mouse events entirely.

## Potential Issues

- **SMP stubs when `SMP` undefined:** Functions like `GLimp_SpawnRenderThread` are no-ops when compiled without `-DSMP`, silently disabling parallelism—could be surprising if linking against a differently-configured renderer DLL.
- **X error handler installation:** `XSetErrorHandler(GLW_ErrorHandler)` is called unconditionally, but error recovery is minimal (just prints).
