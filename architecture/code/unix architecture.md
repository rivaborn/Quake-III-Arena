# Subsystem Overview

## Purpose
The `code/unix` subsystem is the Linux/Unix platform layer for Quake III Arena, providing all OS-specific implementations required by the engine's platform-agnostic core. It bridges the engine's abstract `Sys_*`, `GLimp_*`, `IN_*`, and `SNDDMA_*` interfaces to Linux kernel, X11, POSIX, and OSS facilities. It covers display initialization, input handling, audio output, networking, filesystem utilities, signal handling, and the process entry point.

## Key Files

| File | Role |
|---|---|
| `unix_main.c` | Process entry point (`main`), engine init/frame loop, TTY console, `Sys_*` shutdown/error, DLL loading via `dlopen`, system event queue |
| `linux_glimp.c` | X11 window creation, GLX context management, video mode switching, gamma, mouse/keyboard grab, SMP render thread, `GLimp_*` and `IN_*` entry points |
| `linux_qgl.c` | Runtime OpenGL library loading (`dlopen`/`dlsym`), `qgl*` function pointer table population, optional per-call GL logging |
| `linux_snd.c` | OSS `/dev/dsp` DMA audio driver; implements `SNDDMA_*` interface for the portable mixing layer |
| `unix_net.c` | UDP socket creation, packet send/receive, local address enumeration, LAN classification; implements `Sys_*`/`NET_*` network API |
| `unix_shared.c` | Millisecond timer, directory/file listing, path resolution, user and CPU queries; implements POSIX `Sys_*` utilities |
| `linux_joystick.c` | Linux kernel joystick device polling (`/dev/jsN`), axis/button-to-key-event translation |
| `linux_signals.c` | POSIX signal handler installation (`InitSig`), graceful shutdown with optional `GLimp_Shutdown` call |
| `linux_common.c` | Platform overrides for `Com_Memcpy`/`Com_Memset`; contains disabled MMX/x86 fast-path assembly |
| `linux_local.h` | Internal platform header; declares all inter-module contracts for the Unix layer |
| `unix_glw.h` | Declares `glwstate_t` and the `glw_state` extern shared by the Unix GL modules |
| `vm_x86.c` | No-op stub for `VM_Compile`/`VM_CallCompiled`; forces the Unix build to fall back to interpreted Q3VM |
| `qasm.h` | Shared header for `.s`/`.nasm` assembly files; defines `C()` name-mangling macro, struct byte offsets, and `.extern` declarations for software-renderer and audio globals |

## Core Responsibilities

- Implement the OS entry point, main loop, and all `Sys_*` functions required by `qcommon`
- Create and manage the X11/GLX window, rendering context, and display gamma
- Dynamically load the OpenGL shared library and populate the `qgl*` function pointer table
- Handle all X11 input events (keyboard, mouse via DGA or relative motion) and joystick devices
- Drive DMA audio output through the OSS interface, including mmap ring-buffer management
- Provide UDP networking (socket lifecycle, packet I/O, local address enumeration)
- Install POSIX signal handlers for graceful process shutdown
- Supply platform-specific `Com_Memcpy`/`Com_Memset` overrides and assembly-level struct offset definitions
- Stub out the x86 JIT compiler, directing all VM execution to the interpreted path

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `Sys_*` functions (`Sys_GetEvent`, `Sys_QueEvent`, `Sys_LoadDll`, `Sys_Error`, `Sys_Quit`, `Sys_Milliseconds`, `Sys_Mkdir`, `Sys_ListFiles`, `Sys_GetCurrentUser`) — consumed by `qcommon`
- `GLimp_*` entry points (`GLimp_Init`, `GLimp_Shutdown`, `GLimp_EndFrame`, `GLimp_SetGamma`, `GLimp_SpawnRenderThread`) — consumed by `code/renderer`
- `IN_Init`, `IN_Shutdown`, `IN_Frame` — consumed by `code/client`
- `SNDDMA_*` interface — consumed by `code/client/snd_dma.c`
- `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` — consumed by `linux_glimp.c` and renderer init
- `InitSig` — called during engine startup from `unix_main.c`
- `Com_Memcpy`, `Com_Memset` — override symbols consumed engine-wide via `qcommon/common.c`
- `glw_state` (`glwstate_t`) — shared state struct used across `linux_glimp.c` and `linux_qgl.c`

**Consumed from other subsystems:**
- `Com_Init`, `Com_Frame` — engine core loop (`qcommon/common.c`)
- `CL_Shutdown`, `NET_Init` — client and network layers
- `Sys_SendKeyEvents` — client input dispatcher
- `dma` global (`dma_t`) — defined in `snd_dma.c`; `linux_snd.c` writes into it directly
- `glConfig`, `glState`, `ri` (refimport), all `r_*` cvars — renderer globals consumed by `linux_glimp.c` and `linux_qgl.c`
- `saved_euid` — defined in `unix_main.c`; consumed by `linux_snd.c` and `linux_qgl.c` for privilege escalation checks
- `in_joystick`, `in_joystickDebug`, `joy_threshold` — CVARs registered in `linux_glimp.c`, consumed by `linux_joystick.c`
- `Sys_XTimeToSysTime`, `Sys_Milliseconds` — defined in `unix_shared.c`, called from `linux_glimp.c`

## Runtime Role

**Init:**
- `unix_main.c:main()` calls `Com_Init` to bring up `qcommon`, then calls `IN_Init` (which triggers `linux_glimp.c` to open the X11 display, create the GLX context, and call `QGL_Init` to load the OpenGL library and resolve function pointers) and `SNDDMA_Init` (which opens the OSS device and maps the DMA buffer). `InitSig` is called to install POSIX signal handlers. `IN_StartupJoystick` opens `/dev/jsN` if a joystick is present.

**Per-frame:**
- `unix_main.c` drives `Com_Frame`; within each frame, `IN_Frame` calls `HandleEvents` to drain the X11 event queue and `IN_JoyMove` to poll joystick state. `GLimp_EndFrame` calls `glXSwapBuffers`. `SNDDMA_GetDMAPos` queries the OSS playback pointer. `unix_net.c` sends/receives UDP packets as driven by `qcommon`.

**Shutdown:**
- `Sys_Quit`/`Sys_Error` in `unix_main.c` call `CL_Shutdown` (which invokes `GLimp_Shutdown` to destroy the GLX context, close the X11 display, unload the OpenGL library, and restore video mode/gamma) and `SNDDMA_Shutdown` (which unmaps the DMA buffer and closes the OSS device). On fatal signal, `linux_signals.c` calls `GLimp_Shutdown` before delegating to `Sys_Exit`.

## Notable Implementation Details

- **SMP rendering:** `linux_glimp.c` supports an optional dedicated render thread via pthreads (`GLimp_SpawnRenderThread`), synchronized with semaphores — an uncommon feature for a 1999-era game engine.
- **Disabled MMX path:** `linux_common.c` contains a fully written but `#if 0`-guarded MMX `memset` and hand-unrolled scalar `memcpy`; the active build simply calls libc.
- **Dual-mode GL dispatch:** `linux_qgl.c` maintains two function pointer tables (`qgl*` and `dll*`): the `dll*` table always holds the raw library pointers, while `qgl*` is swapped between direct and logging (`log*`) wrappers at runtime without reloading the library.
- **Setuid privilege escalation:** Both `linux_snd.c` and `linux_qgl.c` check `saved_euid` (captured at startup in `unix_main.c`) and use `seteuid` to temporarily elevate privileges when opening `/dev/dsp` or searching CWD for the OpenGL library.
- **JIT stub:** `vm_x86.c` is a deliberate no-op, meaning all QVM bytecode on Linux/Unix runs through the interpreter in `qcommon/vm_interpreted.c`. The Windows-side `code/win32/` contains the actual x86 JIT.
- **`qasm.h` layout coupling:** The struct byte-offset constants in `qasm.h` must remain manually synchronized with their C counterparts in `model.h`, `sound.h`, `r_local.h`, etc.; there is no compile-time enforcement of this invariant.
- **DGA vs. relative mouse:** `linux_glimp.c` supports both XFree86-DGA (direct hardware mouse access) and standard X11 relative motion events as fallback, selected at runtime.
- **glibc `memset` workaround:** `linux_snd.c` defines a local `Snd_Memset` to work around a known glibc bug that corrupts the mmap'd DMA buffer on certain kernel/libc versions.
