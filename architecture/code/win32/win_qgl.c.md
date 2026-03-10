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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `glwstate_t` | struct (defined in `glw_win.h`) | Holds `hinstOpenGL` (DLL handle), `log_fp` (log file pointer), WGL context handles |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `qgl*` (e.g. `qglBegin`) | global function pointers (~245) | global | Active GL entry points used by all renderer code |
| `qwgl*` (e.g. `qwglMakeCurrent`) | global function pointers (~20) | global | Active WGL entry points |
| `dll*` (e.g. `dllBegin`) | static function pointers (~245) | static | Immutable DLL-resolved pointers; restored when logging disabled |
| `glw_state` | `glwstate_t` | global (defined in `win_glimp.c`) | Holds `hinstOpenGL`, `log_fp`, and other Win32 GL window state |

## Key Functions / Methods

### QGL_Init
- **Signature:** `qboolean QGL_Init( const char *dllname )`
- **Purpose:** Loads the OpenGL DLL and binds all `qgl*`/`dll*`/`qwgl*` function pointers. Also initialises extension pointers to 0 and calls `QGL_EnableLogging`.
- **Inputs:** `dllname` — name of the OpenGL DLL (e.g. `"opengl32"` or a 3Dfx path)
- **Outputs/Return:** `qtrue` on success, `qfalse` if the DLL cannot be loaded or Glide is missing
- **Side effects:** Calls `LoadLibrary`; populates ~265 global function pointers; may open a log file
- **Calls:** `GetSystemDirectory`, `LoadLibrary`, `GPA` (macro → `GetProcAddress`), `GlideIsValid`, `QGL_EnableLogging`, `ri.Printf`, `Com_sprintf`, `Q_strncpyz`
- **Notes:** Asserts `glw_state.hinstOpenGL == 0` before loading. The `dllname` must be lower-case per comment.

### QGL_Shutdown
- **Signature:** `void QGL_Shutdown( void )`
- **Purpose:** Unloads the OpenGL DLL and NULLs every `qgl*` and `qwgl*` pointer to prevent stale calls.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `FreeLibrary`; sets `glw_state.hinstOpenGL = NULL`; zeroes ~265 global pointers
- **Calls:** `FreeLibrary`, `ri.Printf`
- **Notes:** Called during hard renderer restarts (`vid_restart`).

### QGL_EnableLogging
- **Signature:** `void QGL_EnableLogging( qboolean enable )`
- **Purpose:** Toggles the logging interception layer. When enabled, swaps every `qgl*` pointer to the corresponding `log*` wrapper; when disabled, restores the direct `dll*` pointers.
- **Inputs:** `enable` — whether to activate logging
- **Outputs/Return:** void
- **Side effects:** Opens/closes `gl.log` via `fopen`/`fclose`; reassigns ~245 `qgl*` pointers; decrements `r_logFile` cvar counter each call when already active
- **Calls:** `ri.Cvar_Set`, `ri.Cvar_Get`, `time`, `localtime`, `asctime`, `fopen`, `fclose`, `fprintf`, `Com_sprintf`
- **Notes:** Uses a `static qboolean isEnabled` guard to avoid redundant swaps. The log file is opened at `<fs_basepath>/gl.log`.

### GlideIsValid
- **Signature:** `static qboolean GlideIsValid( void )`
- **Purpose:** Checks whether a valid 3Dfx Glide3X DLL exists on the system.
- **Inputs:** None
- **Outputs/Return:** `qtrue` if `Glide3X` can be loaded (always returns `qtrue` if present — the board-count check is `#if 0`'d out)
- **Side effects:** Calls `LoadLibrary("Glide3X")` — the loaded handle is never freed (bug in current code)
- **Notes:** The board-count validation path is disabled; `FIXME` comment references 3Dfx.

### log* wrappers (e.g. `logBegin`, `logDrawElements`, `logViewport`)
- ~245 static `APIENTRY` functions, one per GL entry point
- Each writes the call name (and selected parameters) to `glw_state.log_fp` via `fprintf`/`SIG`, then forwards to the corresponding `dll*` pointer
- Notable detail: some wrappers decode enum values to strings using helpers `BooleanToString`, `FuncToString`, `PrimToString`, `CapToString`, `TypeToString`, `BlendToName`

## Control Flow Notes
- **Init:** `QGL_Init` is called from `GLimp_Init` (in `win_glimp.c`) at renderer startup. It must succeed before any GL call is made.
- **Per-frame:** `qgl*` pointers are called directly by the renderer backend throughout every frame. No per-frame work occurs in this file itself.
- **Logging toggle:** `QGL_EnableLogging` is called from `QGL_Init` and may be re-invoked mid-session via the `r_logFile` cvar.
- **Shutdown:** `QGL_Shutdown` is called from `GLimp_Shutdown` during renderer teardown or `vid_restart`.

## External Dependencies
- `#include <float.h>` — standard C
- `#include "../renderer/tr_local.h"` — provides `ri` (refimport), `r_logFile` cvar, `glconfig_t`, renderer types
- `#include "glw_win.h"` — provides `glwstate_t` and `glw_state` (the Win32 GL window/context state)
- **Defined elsewhere:** `glw_state` (defined in `win_glimp.c`); `ri` (renderer import table); `r_logFile`, `qglActiveTextureARB`, `qglClientActiveTextureARB`, `qglMultiTexCoord2fARB`, `qglLockArraysEXT`, `qglUnlockArraysEXT` (declared/used in renderer modules); Windows API: `LoadLibrary`, `FreeLibrary`, `GetProcAddress`, `GetSystemDirectory`
