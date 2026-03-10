# code/unix/unix_main.c — Enhanced Analysis

## Architectural Role
This file implements the complete Unix/Linux platform abstraction layer (`Sys_*` interface) and is the primary entry point (`main`) for the engine. It sits at the boundary between the C runtime and the engine core (`qcommon`), responsible for: (1) initializing and shutting down platform-specific services (TTY, signals, DLLs, privilege), (2) polling all input sources (keyboard, console, network, joystick) and funneling them through a unified event queue for frame-synchronous consumption by `Com_Frame`, and (3) dynamically loading the game/cgame/ui VMs as native `.so` modules. It is the sole platform-layer file for dedicated and client builds on Linux/Unix systems (complementary to `win32/win_main.c` on Windows, `macosx/macosx_sys.m` on macOS).

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon/common.c**: `Com_Frame` calls `Sys_GetEvent` every frame to drain the input event queue; this is the primary frame-loop integration point.
- **qcommon/vm.c**: `VM_LoadNative` calls `Sys_LoadDll` to load native game/cgame/ui modules at VM initialization time.
- **qcommon/common.c**: `Com_Error` may indirectly call `Sys_Error` if the error handler needs to shut down (though the typical path is `CL_Shutdown` → `Sys_Exit`).
- **C runtime**: `main()` is the kernel-called entry point; the frame loop never returns (infinite `while(1)` pattern).

### Outgoing (what this file depends on)
- **qcommon/common.c**: `Com_Init`, `Com_Frame`, `Cvar_Get`, `Cvar_Set`, `Cmd_AddCommand`, `Com_Printf`, `Com_Error`, `Hunk_*` memory allocation.
- **qcommon/files.c**: `FS_BuildOSPath`, `FS_Read`, `FS_Seek` for virtual filesystem queries during DLL path resolution.
- **qcommon/net_chan.c & qcommon/msg.c**: `Sys_GetPacket` polling; `MSG_Init`, Huffman decompression (called from `Sys_GetEvent` before queuing network packets).
- **client/cl_main.c**: `CL_Shutdown` called on error/quit paths; also reads `clc.demofile` and other client state during main loop.
- **client/cl_input.c**: `Sys_SendKeyEvents` called from `Sys_GetEvent` to pump raw keyboard state into the input subsystem.
- **qcommon/common.h**: Global `re` (refexport_t) export variable—read by renderer initialization but set by loading the renderer DLL (in non-dedicated builds this would be done elsewhere; on Unix the pattern varies by build mode).
- **renderer/tr_public.h**: Definitions for `refexport_t` vtable.
- **Sys_***: Global integers `sys_frame_time` and `stdin_active` are read/written by platform-layer code and external modules.

## Design Patterns & Rationale

**Event Queue Pattern** (circular buffer `eventQue[MAX_QUED_EVENTS]`): Decouples input producers (keyboard reader, network socket, joystick, console) from the consumer (`Com_Frame`). All input is queued as `sysEvent_t` records with timestamps, allowing the frame loop to process input synchronously without blocking I/O. This is fundamental to id Tech 3's deterministic frame-driven architecture; modern engines use event callbacks or async I/O instead.

**Platform Abstraction Layer**: All Unix-specific APIs (signals, TTY, dlopen, fcntl, termios, pthreads) are hidden behind a platform-agnostic `Sys_*` interface. Windows (`win32/win_main.c`) and macOS (`macosx/macosx_sys.m`) reimplement the same signatures, allowing the engine core (`qcommon`, `client`, `server`, `game`) to be platform-agnostic.

**Privilege Dropping**: `main()` immediately calls `seteuid(getuid())` to drop SUID privilege, protecting against accidental execution of malicious code with elevated privileges. The saved `euid` is retained but not used in this file (likely for resource cleanup or file operations in other platform layers).

**Manual TTY Console** (raw mode + line editing): Rather than depend on GNU readline or libedit, this file implements a minimal TTY console: `Sys_ConsoleInputInit` puts stdin into raw mode (`tc.c_lflag &= ~(ECHO | ICANON)`), then `Sys_ConsoleInput` implements backspace, VT100 arrow key handling, tab completion, and command history inline. This is a pre-readline approach; modern ports would likely use ncurses or ncurses-like libraries. The `ttycon_hide/ttycon_show` nesting is necessary because Sys_Printf output during line editing must temporarily hide/restore the line buffer to stdout.

**Native DLL Loading with Architecture Suffix**: `Sys_LoadDll` searches for module variants by architecture macro (`#if defined __i386__` → `"_i386.so"`, etc.) determined at compile time. This avoids runtime architecture detection but is brittle: if compiled as 32-bit but run on a 64-bit system, the 64-bit `.so` is not available. Modern practice uses universal binaries or runtime CPU detection.

## Data Flow Through This File

**Inbound**: Raw bytes from stdin (keyboard in TTY mode, or piped commands in dedicated server mode), UDP network packets (via platform socket layer), joystick events (via `IN_Frame` callback), and internally-generated console commands → queued into `eventQue[MAX_QUED_EVENTS]` with timestamps as `sysEvent_t` records.

**Frame Loop Integration**: Each call to `Com_Frame` (from infinite loop in `main`) triggers `Sys_GetEvent`, which drains the event queue one event at a time. On first call (empty queue), all input sources are polled: `Sys_SendKeyEvents` updates keyboard state in `kbuttons[]`, `Sys_ConsoleInput` reads one character and queues completed lines, `IN_Frame` processes joystick, `Sys_GetPacket` dequeues one UDP packet. Subsequent calls return queued events until empty.

**Outbound**: Engine calls `Sys_LoadDll` to load `cgame_<arch>.so` / `game_<arch>.so` / `ui_<arch>.so`, receiving a dlopen handle and calling `dllEntry(engine_syscalls_function_pointer)` to wire up the engine's syscall dispatcher. On shutdown, `Sys_Error` or `Sys_Quit` triggers `CL_Shutdown` → `Sys_Exit`, restoring TTY attributes via `Sys_ConsoleInputShutdown` and exiting with `_exit` (release) or `exit` (debug).

## Learning Notes

This file is a **time capsule of 2000s-era game engine porting practices**:

- **Event Queue over Callbacks**: The synchronous event queue is the canonical id Tech 3 pattern. Modern engines (Unreal, Unity, Godot) use event callbacks, input handling systems, or async I/O to avoid polling. The queue pattern is still used in some resource-constrained or deterministic systems (roguelikes, turn-based games).

- **Architecture Detection at Compile Time**: The `#if defined __i386__` / `#elif defined __alpha__` blocks bake the architecture into the binary. Modern practice is runtime detection (`__builtin_cpu*` GCC builtins, `/proc/cpuinfo` parsing, or `sysconf`). This approach made sense when Q3A targeted Alpha, MIPS, and Sparc, but it requires separate builds for each architecture.

- **Manual Line Editing**: The TTY console's raw-mode line editor is instructive for understanding how shells work (VT100 escape codes, terminal state management, history navigation) but would never be built this way today. It also demonstrates the complexity of correct Unicode/UTF-8 handling in a simple line editor (no explicit UTF-8 support here; assumes ASCII).

- **DLL Entry Point Convention**: `dllEntry(engine_syscalls)` called immediately after `dlopen` is a lightweight plugin pattern. Modern C++ plugins use vtable classes or exported factory functions; modern Rust plugins use `cdylib` crates with versioned ABI contracts.

- **Security via Privilege Drop**: The SUID pattern (drop privilege immediately after gaining it for resource access) is still sound, but the code doesn't actually use elevated privileges—it's a defensive measure against bugs. Modern practice is to not use SUID at all; use systemd socket activation or capabilities (`setcap`) instead.

## Potential Issues

- **Silent Event Queue Overflow** (line ~910 in full file): When `eventQue` fills, the oldest event is freed and discarded without warning. High-frequency input (network packets during demo playback) can silently lose events, causing missed keypresses or desync. Modern event queues either block or grow dynamically.

- **TTY Console Not Thread-Safe**: If the renderer or networking code ever calls `Sys_Printf` from a background thread (SMP renderer front-end/back-end), the `ttycon_hide/show` nesting counter and `tty_con.buffer` access are races. A mutex would be required.

- **No DLL Validation** (line ~1050): `Sys_LoadDll` calls `dlsym(handle, "dllEntry")` and immediately calls the resolved function pointer without checking for NULL. If the `.so` is corrupted or from a different version, this will segfault. Modern plugin loaders validate symbol presence and version before calling.

- **FPU State Configuration Inconsistency**: `Sys_ConfigureFPU` is called only in `DEDICATED` builds (in `main`), but non-dedicated client/listen-server builds may not initialize FPU control state. The renderer GLimp layer is supposed to call it, but if the renderer isn't loaded or fails, FPU exceptions (divide-by-zero) may not be caught.

- **Unbounded `cmdline` Allocation** (line ~750): All command-line arguments are concatenated into a single malloc'd string with no length check. A pathological invocation with many/long arguments could overflow or exhaust memory before `Com_Init` is even called.
