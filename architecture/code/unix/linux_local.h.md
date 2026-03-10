# code/unix/linux_local.h

## File Purpose
Platform-specific header for the Linux port of Quake III Arena, declaring all Linux/Unix-specific subsystem interfaces. It serves as the internal contract between the Linux platform layer modules (input, GL, signals, system events).

## Core Responsibilities
- Declare system event queue injection interface
- Declare input subsystem lifecycle and per-frame functions
- Declare joystick startup and polling functions
- Declare OpenGL dynamic library (QGL) management interface
- Declare signal handler initialization
- Provide a `strlwr` utility absent from standard POSIX libc

## Key Types / Data Structures
None. (All types used — `sysEventType_t`, `netadr_t`, `msg_t`, `qboolean` — are defined elsewhere.)

## Global / File-Static State
None.

## Key Functions / Methods

### Sys_QueEvent
- **Signature:** `void Sys_QueEvent(int time, sysEventType_t type, int value, int value2, int ptrLength, void *ptr)`
- **Purpose:** Enqueues a system event (keyboard, mouse, network, etc.) into the engine's event queue.
- **Inputs:** `time` — event timestamp; `type` — event category; `value`/`value2` — event-specific integers; `ptrLength`/`ptr` — optional payload.
- **Outputs/Return:** None.
- **Side effects:** Mutates the global system event ring buffer (defined in `qcommon`).
- **Calls:** Defined elsewhere (qcommon/sys layer).
- **Notes:** Central dispatch point; all platform input paths funnel through here.

### Sys_GetPacket
- **Signature:** `qboolean Sys_GetPacket(netadr_t *net_from, msg_t *net_message)`
- **Purpose:** Reads one pending UDP packet from the network socket into `net_message`, filling `net_from` with the sender address.
- **Inputs:** Pointers to caller-allocated address and message buffers.
- **Outputs/Return:** `qtrue` if a packet was available, `qfalse` otherwise.
- **Side effects:** Reads from OS socket; fills output buffers.
- **Calls:** Defined in `unix_net.c` or similar.
- **Notes:** Called per-frame by the network layer.

### IN_Init / IN_Frame / IN_Shutdown
- **Signature:** `void IN_Init(void)` / `void IN_Frame(void)` / `void IN_Shutdown(void)`
- **Purpose:** Lifecycle management for the Linux input subsystem (mouse, keyboard, joystick).
- **Side effects:** `IN_Init` opens input devices; `IN_Frame` pumps X11/evdev events and calls `Sys_QueEvent`; `IN_Shutdown` closes devices.
- **Calls:** `IN_Frame` typically calls `IN_JoyMove`, X11 event polling, `Sys_QueEvent`.

### QGL_Init / QGL_EnableLogging / QGL_Shutdown
- **Signature:** `qboolean QGL_Init(const char *dllname)` / `void QGL_EnableLogging(qboolean enable)` / `void QGL_Shutdown(void)`
- **Purpose:** Dynamically load/unload the OpenGL shared library (`libGL.so`) and resolve all GL function pointers; optionally wrap them with logging stubs.
- **Side effects:** `QGL_Init` calls `dlopen`; `QGL_Shutdown` calls `dlclose`; `QGL_EnableLogging` swaps function pointer table.
- **Calls:** Defined in `linux_qgl.c`.

### InitSig
- **Signature:** `void InitSig(void)`
- **Purpose:** Installs POSIX signal handlers (SIGSEGV, SIGTERM, etc.) for crash reporting and clean shutdown.
- **Side effects:** Modifies process signal disposition via `sigaction`.
- **Calls:** Defined in `linux_signals.c`.

### strlwr
- **Signature:** `char *strlwr(char *s)`
- **Purpose:** In-place ASCII lowercase conversion; POSIX does not provide `strlwr` (it is a MSVC extension).
- **Inputs:** Mutable C string.
- **Outputs/Return:** Same pointer `s`.

## Control Flow Notes
Included by Linux platform `.c` files during init: `IN_Init` and `QGL_Init` are called from renderer/client init; `IN_Frame` is called each client frame before event dispatch; `InitSig` is called once at process startup from `unix_main.c`. `Sys_QueEvent` is called throughout the frame from input and network polling paths.

## External Dependencies
- `q_shared.h` / `qcommon.h` — `qboolean`, `sysEventType_t`, `netadr_t`, `msg_t` (defined elsewhere)
- `dlopen`/`dlclose` — used by `QGL_Init`/`QGL_Shutdown` implementations (glibc `<dlfcn.h>`)
- POSIX `<signal.h>` — used by `InitSig` implementation
