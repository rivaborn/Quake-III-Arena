# code/unix/linux_signals.c

## File Purpose
Installs POSIX signal handlers for the Linux build of Quake III Arena, enabling graceful shutdown on fatal or termination signals. It guards against double-signal re-entry and optionally shuts down the OpenGL renderer before exiting.

## Core Responsibilities
- Register a unified `signal_handler` for all critical POSIX signals via `InitSig`
- Detect double-signal re-entry using a static flag and force-exit in that case
- Shut down the OpenGL/renderer subsystem (`GLimp_Shutdown`) on the first signal (non-dedicated build only)
- Delegate final process exit to `Sys_Exit` rather than calling `exit()` directly

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `signalcaught` | `static qboolean` | file-static | Re-entrancy guard; set on first signal, triggers hard exit on second |

## Key Functions / Methods

### signal_handler
- **Signature:** `static void signal_handler(int sig)`
- **Purpose:** Unified handler for all registered signals; performs a clean or forced shutdown depending on whether a signal has already been caught.
- **Inputs:** `sig` — the signal number delivered by the OS.
- **Outputs/Return:** `void`; never returns (always calls `Sys_Exit`).
- **Side effects:** Writes `signalcaught = qtrue`; prints to `stdout`; calls `GLimp_Shutdown` (non-dedicated); calls `Sys_Exit`.
- **Calls:** `printf`, `GLimp_Shutdown` (conditional on `!DEDICATED`), `Sys_Exit`
- **Notes:** On second entry (double-fault), exits with code `1`. On first entry, exits with code `0`. The code comment questions whether `CL_Shutdown` should be called instead of `GLimp_Shutdown`.

### InitSig
- **Signature:** `void InitSig(void)`
- **Purpose:** Registers `signal_handler` for all signals the engine wants to intercept.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Modifies the process's signal disposition table via `signal()`.
- **Calls:** `signal` (9 times, covering `SIGHUP`, `SIGQUIT`, `SIGILL`, `SIGTRAP`, `SIGIOT`, `SIGBUS`, `SIGFPE`, `SIGSEGV`, `SIGTERM`)
- **Notes:** `SIGINT` is notably absent — Ctrl-C is not intercepted here.

## Control Flow Notes
`InitSig` is called once during engine startup (from `unix_main.c` or equivalent platform init). Thereafter the handler is purely reactive, triggered asynchronously by the OS on signal delivery. It sits outside the normal frame loop and is only invoked on abnormal or termination events.

## External Dependencies
- `<signal.h>` — POSIX signal API (`signal`, `SIGHUP`, `SIGQUIT`, etc.)
- `../game/q_shared.h` — `qboolean`, `qfalse`, `qtrue`
- `../qcommon/qcommon.h` — (included for shared definitions; no direct calls visible here)
- `../renderer/tr_local.h` — `GLimp_Shutdown` (included only when `DEDICATED` is not defined)
- `Sys_Exit` — declared via forward declaration (`void Sys_Exit(int)`); defined in `unix_main.c`
- `GLimp_Shutdown` — defined in `linux_glimp.c`
