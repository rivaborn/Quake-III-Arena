# code/unix/linux_qgl.c

## File Purpose
Implements the Linux/Unix operating system binding of OpenGL to QGL function pointers by dynamically loading an OpenGL shared library via `dlopen`/`dlsym`. It provides a thin indirection layer with two modes: direct dispatch (pointers point straight to the loaded library symbols) and logging dispatch (pointers point to `log*` wrappers that write to a file before forwarding to the real function).

## Core Responsibilities
- Load the OpenGL shared library at runtime using `dlopen`
- Resolve all ~230 OpenGL 1.1 entry points plus GLX and optional extension functions via `dlsym` (macro `GPA`)
- Expose the resolved addresses through the global `qgl*` function pointer table consumed by the rest of the renderer
- Maintain a parallel `dll*` shadow table holding the raw library addresses
- Provide per-call GL logging (writes function name/args to `gl.log`) by swapping `qgl*` pointers to `log*` wrappers
- Null out all `qgl*` pointers and close the library handle on shutdown

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `glwstate_t` | struct (defined in `unix_glw.h`) | Holds the `dlopen` handle (`OpenGLLib`) and the log file pointer (`log_fp`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `glw_state` | `glwstate_t` | global (defined elsewhere, extern here) | Shared state: open GL library handle and log file |
| `qgl*` (≈230 entries) | function pointers | global | Active dispatch table used by the renderer |
| `dll*` (≈230 entries) | function pointers (most `static`) | file-static | Preserved raw addresses from the loaded library; `qgl*` is restored to these when logging is disabled |
| `qglX*` / `qfxMesa*` | function pointers | global | GLX context functions and optional FX Mesa context functions |

## Key Functions / Methods

### QGL_Init
- **Signature:** `qboolean QGL_Init( const char *dllname )`
- **Purpose:** Loads the named OpenGL shared library and resolves all GL, GLX, and extension entry points into both the `qgl*` and `dll*` tables.
- **Inputs:** `dllname` — path/name of the OpenGL `.so` to load (e.g., `libGL.so.1`)
- **Outputs/Return:** `qtrue` on success, `qfalse` if the library cannot be opened
- **Side effects:** Mutates all `qgl*` and `dll*` globals; calls `dlopen`; calls `ri.Printf` on failure; reads `saved_euid` (extern from `unix_main.c`) to decide whether to try the current working directory as a fallback
- **Calls:** `dlopen`, `GPA` (macro → `dlsym`), `getcwd`, `Q_strcat`, `ri.Printf`
- **Notes:** Extension pointers (`qglLockArraysEXT`, `qglActiveTextureARB`, etc.) are explicitly zeroed after base resolution; callers must use `qwglGetProcAddress` or platform-specific extension queries to populate them.

### QGL_Shutdown
- **Signature:** `void QGL_Shutdown( void )`
- **Purpose:** Closes the OpenGL library handle and nulls all `qgl*` function pointers, including GLX and optional FX Mesa entries.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `dlclose`; sets `glw_state.OpenGLLib = NULL`; zeroes ~230 global function pointers
- **Calls:** `dlclose`

### QGL_EnableLogging
- **Signature:** `void QGL_EnableLogging( qboolean enable )`
- **Purpose:** Toggles per-call GL logging. When enabled, all `qgl*` pointers are redirected to `log*` wrappers that write to `gl.log` then forward to the real function. When disabled, `qgl*` is restored to `dll*`.
- **Inputs:** `enable` — non-zero to activate logging, zero to deactivate
- **Outputs/Return:** void
- **Side effects:** Opens `<fs_basepath>/gl.log` on first activation; decrements `r_logFile` cvar each call while enabled, stopping when it reaches 0; rewrites all ~230 `qgl*` pointers
- **Calls:** `fopen`, `time`, `localtime`, `asctime`, `fprintf`, `ri.Cvar_Set`, `ri.Cvar_Get`, `va`, `Com_sprintf`, `ri.Printf`
- **Notes:** Uses a file-static `isEnabled` to avoid redundant work; the countdown behavior via `r_logFile` allows capturing exactly N frames of GL calls.

### qwglGetProcAddress
- **Signature:** `void *qwglGetProcAddress( char *symbol )`
- **Purpose:** Provides a `wglGetProcAddress`-compatible interface for resolving extension entry points from the already-open GL library handle.
- **Inputs:** `symbol` — GL extension function name string
- **Outputs/Return:** function pointer or `NULL` if library not loaded
- **Side effects:** None
- **Calls:** `dlsym` (via `GPA` macro)

### GLimp_LogNewFrame
- **Signature:** `void GLimp_LogNewFrame( void )`
- **Purpose:** Writes a frame-boundary marker (`*** R_BeginFrame ***`) to the GL log file.
- **Side effects:** I/O to `glw_state.log_fp`

### Notes
- The ~230 `log*` wrapper functions are trivial: each calls `fprintf(glw_state.log_fp, ...)` then delegates to the corresponding `dll*` pointer. They are not individually documented here.

## Control Flow Notes
- **Init:** Called from `linux_glimp.c` → `GLimp_Init` during renderer startup; `QGL_Init` must succeed before any GL call can be made.
- **Per-frame:** If logging is active, `GLimp_LogNewFrame` is called at the start of each rendered frame to delimit log output.
- **Shutdown:** `QGL_Shutdown` is called from `GLimp_Shutdown` during renderer teardown, before the window/display is destroyed.
- The file has no frame-update logic of its own; it is purely a dispatch-table manager.

## External Dependencies
- `<dlfcn.h>` — `dlopen`, `dlclose`, `dlsym`, `dlerror`
- `<unistd.h>` — `getcwd`, `getuid`
- `../renderer/tr_local.h` — renderer globals (`r_logFile`, `ri`, `glw_state` usage context), `qboolean`, `Q_strcat`, `Com_sprintf`, `ri.Printf`, `ri.Cvar_*`
- `unix_glw.h` — `glwstate_t`, `glw_state` declaration
- `saved_euid` — defined in `code/unix/unix_main.c`; used to detect setuid execution and conditionally try CWD library lookup
- All `qgl*` function pointer declarations consumed by `code/renderer/` subsystem (defined elsewhere, populated here)
