# code/ui/ui_syscalls.c

## File Purpose
Provides the DLL-side system call bridge for the UI module, mapping high-level `trap_*` functions to indexed engine syscalls via a single function pointer. This file is only compiled for DLL builds; the QVM equivalent is `ui_syscalls.asm`.

## Core Responsibilities
- Store and initialize the engine-provided `syscall` function pointer via `dllEntry`
- Wrap every engine service (rendering, sound, cvars, filesystem, networking, input, cinematics) behind typed `trap_*` C functions
- Handle float-to-int reinterpretation via `PASSFLOAT` to safely pass floats through the variadic integer syscall ABI
- Expose CD-key validation and PunkBuster status reporting to the UI module
- Provide LAN/server browser query traps for the multiplayer server list UI

## Key Types / Data Structures
None defined in this file; all types (`vmCvar_t`, `refEntity_t`, `glconfig_t`, etc.) come from included headers.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `syscall` | `static int (QDECL *)( int arg, ... )` | static (file) | Holds the engine-provided syscall dispatcher; initialized to `-1` (invalid) until `dllEntry` is called |

## Key Functions / Methods

### dllEntry
- **Signature:** `void dllEntry( int (QDECL *syscallptr)( int arg, ... ) )`
- **Purpose:** Engine entry point to inject the syscall function pointer into the UI DLL at load time.
- **Inputs:** `syscallptr` — engine's variadic dispatcher
- **Outputs/Return:** void
- **Side effects:** Writes the file-static `syscall` pointer
- **Calls:** None
- **Notes:** Must be called before any `trap_*` function; calling any trap before this causes a crash via the `-1` sentinel pointer.

### PASSFLOAT
- **Signature:** `int PASSFLOAT( float x )`
- **Purpose:** Reinterpret a `float` bit-pattern as `int` so it can be passed through the integer-only variadic syscall boundary without value conversion.
- **Inputs:** `x` — float value
- **Outputs/Return:** The same 32-bit pattern as an `int`
- **Side effects:** None
- **Calls:** None
- **Notes:** Used by every trap that passes a float argument (e.g., `trap_R_DrawStretchPic`, `trap_R_AddLightToScene`). The engine reads it back as a float.

### trap_R_DrawStretchPic
- **Signature:** `void trap_R_DrawStretchPic( float x, float y, float w, float h, float s1, float t1, float s2, float t2, qhandle_t hShader )`
- **Purpose:** Submits a textured quad to the renderer for 2D UI drawing.
- **Inputs:** Screen position/size, UV coordinates, shader handle
- **Outputs/Return:** void
- **Side effects:** Issues `UI_R_DRAWSTRETCHPIC` syscall; renderer queues the draw
- **Calls:** `syscall`, `PASSFLOAT`
- **Notes:** All eight float args are passed through `PASSFLOAT`; most complex syscall in the file in terms of argument count.

### trap_CIN_PlayCinematic / trap_CIN_StopCinematic / trap_CIN_RunCinematic / trap_CIN_DrawCinematic / trap_CIN_SetExtents
- **Signature:** (see file)
- **Purpose:** Full cinematic playback lifecycle — start, advance, draw, resize, and stop FMV sequences from the UI.
- **Side effects:** Cinematic state managed by engine; `trap_CIN_StopCinematic` returns `FMV_EOF`.
- **Notes:** Handles must be stopped in reverse creation order per comment.

### Notes (trivial traps)
- All remaining `trap_*` functions are one-line wrappers: they marshal arguments and forward to `syscall` with the corresponding `UI_*` enum constant.
- LAN traps (`trap_LAN_*`) form a cohesive group for server browsing, ping management, and server list caching.

## Control Flow Notes
`dllEntry` is called once at DLL load by the engine, injecting the syscall pointer. After that, `trap_*` functions are called on-demand throughout the UI frame loop (input, draw, server refresh). There is no per-frame lifecycle within this file itself — it is purely a dispatch layer.

## External Dependencies
- `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`)
- `UI_*` syscall index constants — defined in `ui_public.h` (not in this file)
- All type definitions (`vmCvar_t`, `refEntity_t`, `glconfig_t`, `qtime_t`, `e_status`, `fontInfo_t`, etc.) — defined elsewhere in shared/game/renderer headers
- `QDECL` calling convention macro — defined in `q_shared.h`
