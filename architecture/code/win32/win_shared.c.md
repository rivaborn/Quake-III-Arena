# code/win32/win_shared.c

## File Purpose
Provides Windows-specific implementations of shared system services required by the Quake III engine, including timing, floating-point snapping, CPU feature detection, and user/path queries. This file bridges the platform-agnostic `Sys_*` interface declared in `qcommon.h` to Win32 APIs.

## Core Responsibilities
- Provide `Sys_Milliseconds` using `timeGetTime()` with a stable epoch base
- Implement `Sys_SnapVector` via x86 FPU inline assembly (`fistp`) for fast float-to-int truncation
- Detect CPU capabilities (Pentium, MMX, 3DNow!, KNI/SSE) via CPUID and return a capability constant
- Query the Windows username via `GetUserName`
- Provide default home/install path resolution

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `sys_timeBase` | `int` | global | Timestamp captured at first `Sys_Milliseconds` call; used to normalize all subsequent times to a zero-based epoch |
| `initialized` (in `Sys_Milliseconds`) | `static qboolean` | static (local) | Guards one-time initialization of `sys_timeBase` |
| `s_userName` (in `Sys_GetCurrentUser`) | `static char[1024]` | static (local) | Cached buffer for the Windows username |
| `tmp` (in `fastftol`) | `static int` | static (local) | Scratch register for FPU-to-integer transfer |

## Key Functions / Methods

### Sys_Milliseconds
- **Signature:** `int Sys_Milliseconds(void)`
- **Purpose:** Returns elapsed milliseconds since first call, providing a stable monotonic timer.
- **Inputs:** None.
- **Outputs/Return:** Milliseconds since first invocation as `int`.
- **Side effects:** On first call, sets global `sys_timeBase` via `timeGetTime()`.
- **Calls:** `timeGetTime()` (WinMM).
- **Notes:** Not thread-safe on first initialization; safe thereafter.

### Sys_SnapVector
- **Signature:** `void Sys_SnapVector(float *v)`
- **Purpose:** Snaps each component of a 3-element float vector to the nearest integer using x86 FPU `fistp` (round-to-nearest per FPU rounding mode), used for BSP/network coordinate precision.
- **Inputs:** `v` — pointer to a 3-float array.
- **Outputs/Return:** Modifies `v[0]`, `v[1]`, `v[2]` in place.
- **Side effects:** None beyond modifying `*v`.
- **Calls:** Inline x86 FPU assembly only.
- **Notes:** The `fastftol` alternative is commented out. Behavior depends on the FPU rounding mode (typically round-to-nearest, not truncation).

### Sys_GetProcessorId
- **Signature:** `int Sys_GetProcessorId(void)`
- **Purpose:** Detects the CPU class and returns one of the `CPUID_*` constants for use by the engine to select optimized code paths.
- **Inputs:** None.
- **Outputs/Return:** One of `CPUID_AXP`, `CPUID_GENERIC`, `CPUID_INTEL_UNSUPPORTED`, `CPUID_INTEL_PENTIUM`, `CPUID_AMD_3DNOW`, `CPUID_INTEL_KATMAI`, `CPUID_INTEL_MMX`.
- **Side effects:** None.
- **Calls:** `IsPentium`, `IsMMX`, `Is3DNOW`, `IsKNI`.
- **Notes:** Wrapped in `#pragma optimize("", off/on)` to prevent compiler optimizations from breaking inline CPUID assembly. On non-x86 targets, returns `CPUID_AXP` or `CPUID_GENERIC` via compile-time guards.

### Sys_GetCurrentUser
- **Signature:** `char *Sys_GetCurrentUser(void)`
- **Purpose:** Returns the current Windows login username, falling back to `"player"` on failure or empty result.
- **Inputs:** None.
- **Outputs/Return:** Pointer to static buffer containing the username string.
- **Side effects:** Writes to static `s_userName`.
- **Calls:** `GetUserName` (Win32 API), `strcpy`.
- **Notes:** Returns a static buffer — not re-entrant.

### Sys_DefaultHomePath / Sys_DefaultInstallPath
- **Signature:** `char *Sys_DefaultHomePath(void)` / `char *Sys_DefaultInstallPath(void)`
- **Purpose:** Provide default filesystem paths. Home path is unsupported on Windows (returns `NULL`); install path returns the current working directory.
- **Inputs:** None.
- **Outputs/Return:** `NULL` for home path; `Sys_Cwd()` result for install path.
- **Side effects:** None.
- **Calls:** `Sys_Cwd` (defined elsewhere in `win_main.c`).

#### Notes on static helpers
- `CPUID(func, regs)` — executes raw CPUID opcode via inline asm, populates a 4-register output array; no-ops under `__VECTORC`.
- `IsPentium()` — tests EFLAGS bit 21 (CPUID support) by toggling it.
- `IsMMX()`, `IsKNI()` — check standard CPUID feature bits in EDX (bits 23 and 25 respectively).
- `Is3DNOW()` — queries AMD extended CPUID leaf `0x80000001`, checks EDX bit 31.
- `fastftol(f)` — inline-asm float-to-long via `fistp`; currently unused (its call sites are commented out).

## Control Flow Notes
`Sys_Milliseconds` is called every frame (via `Com_Milliseconds`) to drive game timing. `Sys_GetProcessorId` is called once during `Sys_Init` / engine startup to select code paths. `Sys_SnapVector` is called from physics/network code as needed per frame. The path and user functions are called during filesystem and client initialization.

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `qtrue`/`qfalse`, shared types
- `../qcommon/qcommon.h` — `CPUID_*` constants, `Sys_*` declarations, `Sys_Cwd`
- `win_local.h` — `WinVars_t`, Win32 subsystem headers
- `<windows.h>` (via `win_local.h`) — `GetUserName`
- `<mmsystem.h>` (implicit via WinMM link) — `timeGetTime`
- `Sys_Cwd` — defined elsewhere (not in this file)
