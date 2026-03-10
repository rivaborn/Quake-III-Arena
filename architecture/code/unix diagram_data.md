# code/unix/linux_common.c
## File Purpose
Provides Linux/GAS-syntax x86 assembly implementations of `Com_Memcpy` and `Com_Memset` as drop-in replacements for the MSVC inline-asm versions in `qcommon/common.c`. The active code path (guarded by `#if 1`) simply delegates to libc `memcpy`/`memset`, while the disabled `#else` branch contains hand-optimized MMX/x86 assembly routines.

## Core Responsibilities
- Supply `Com_Memcpy` and `Com_Memset` as Linux platform overrides
- (Disabled) Implement a 32-byte-unrolled scalar x86 `memcpy` with alignment handling
- (Disabled) Implement an MMX-accelerated `memset` via `_copyDWord` for blocks ≥ 8 bytes
- (Disabled) Implement a software prefetch routine `Com_Prefetch` for read/read-write access patterns
- Convert MSVC `__asm` syntax to GAS `__asm__ __volatile__` with local labels and input/output constraints

## External Dependencies
- `<unistd.h>` — for `size_t`
- `<string.h>` — for `memcpy`, `memset` (active path)
- `Com_Prefetch` declared locally (disabled path); defined in same `#else` block
- `_copyDWord` defined locally (disabled path only)
- `Com_Memcpy` / `Com_Memset` symbols consumed by `qcommon/common.c` and the rest of the engine ("defined here, used everywhere")

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

## External Dependencies
- **X11:** `<X11/Xlib.h>` (via GLX), `<X11/keysym.h>`, `<X11/cursorfont.h>`
- **XFree86 extensions:** `<X11/extensions/xf86dga.h>`, `<X11/extensions/xf86vmode.h>`
- **GLX:** `<GL/glx.h>`
- **pthreads:** `<pthread.h>`, `<semaphore.h>`
- **Dynamic linking:** `<dlfcn.h>` — `dlsym` used to resolve ARB extension function pointers from `glw_state.OpenGLLib`
- **Defined elsewhere:** `Sys_QueEvent`, `Sys_XTimeToSysTime`, `Sys_Milliseconds`, `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging`, `InitSig`, `IN_StartupJoystick`, `IN_JoyMove`, `glConfig`, `glState`, `ri` (refimport), `cls`, `com_developer`, all `r_*` cvars, all `q_*` string utilities.

# code/unix/linux_joystick.c
## File Purpose
Implements Linux-specific joystick input handling for Quake III Arena, translating Linux kernel joystick events (`/dev/jsN`) into the engine's internal key event system. It bridges the Linux joystick driver's event model to Quake's polling-style input pipeline.

## Core Responsibilities
- Open and initialize the first available joystick device (`/dev/js0`–`/dev/js3`)
- Drain the joystick event queue each frame
- Dispatch button press/release events directly as `SE_KEY` events
- Convert axis values to a bitmask and synthesize key press/release events for axis transitions
- Map 16 axes to virtual key codes (`joy_keys[]`)

## External Dependencies
- `<linux/joystick.h>` — `struct js_event`, `JS_EVENT_BUTTON`, `JS_EVENT_AXIS`, `JS_EVENT_INIT`, `JSIOCG*` ioctls
- `<fcntl.h>`, `<sys/ioctl.h>`, `<unistd.h>`, `<sys/types.h>` — POSIX I/O
- `../client/client.h` — `cvar_t`, `Com_Printf`, key code constants (`K_LEFTARROW`, `K_JOY1`, etc.)
- `linux_local.h` — `Sys_QueEvent`, `sysEventType_t` (`SE_KEY`)
- `Sys_QueEvent` — defined in `unix_main.c` (not this file)
- `in_joystick`, `in_joystickDebug`, `joy_threshold` — defined/registered in `linux_glimp.c`

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

## External Dependencies
- `q_shared.h` / `qcommon.h` — `qboolean`, `sysEventType_t`, `netadr_t`, `msg_t` (defined elsewhere)
- `dlopen`/`dlclose` — used by `QGL_Init`/`QGL_Shutdown` implementations (glibc `<dlfcn.h>`)
- POSIX `<signal.h>` — used by `InitSig` implementation

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

## External Dependencies
- `<dlfcn.h>` — `dlopen`, `dlclose`, `dlsym`, `dlerror`
- `<unistd.h>` — `getcwd`, `getuid`
- `../renderer/tr_local.h` — renderer globals (`r_logFile`, `ri`, `glw_state` usage context), `qboolean`, `Q_strcat`, `Com_sprintf`, `ri.Printf`, `ri.Cvar_*`
- `unix_glw.h` — `glwstate_t`, `glw_state` declaration
- `saved_euid` — defined in `code/unix/unix_main.c`; used to detect setuid execution and conditionally try CWD library lookup
- All `qgl*` function pointer declarations consumed by `code/renderer/` subsystem (defined elsewhere, populated here)

# code/unix/linux_signals.c
## File Purpose
Installs POSIX signal handlers for the Linux build of Quake III Arena, enabling graceful shutdown on fatal or termination signals. It guards against double-signal re-entry and optionally shuts down the OpenGL renderer before exiting.

## Core Responsibilities
- Register a unified `signal_handler` for all critical POSIX signals via `InitSig`
- Detect double-signal re-entry using a static flag and force-exit in that case
- Shut down the OpenGL/renderer subsystem (`GLimp_Shutdown`) on the first signal (non-dedicated build only)
- Delegate final process exit to `Sys_Exit` rather than calling `exit()` directly

## External Dependencies
- `<signal.h>` — POSIX signal API (`signal`, `SIGHUP`, `SIGQUIT`, etc.)
- `../game/q_shared.h` — `qboolean`, `qfalse`, `qtrue`
- `../qcommon/qcommon.h` — (included for shared definitions; no direct calls visible here)
- `../renderer/tr_local.h` — `GLimp_Shutdown` (included only when `DEDICATED` is not defined)
- `Sys_Exit` — declared via forward declaration (`void Sys_Exit(int)`); defined in `unix_main.c`
- `GLimp_Shutdown` — defined in `linux_glimp.c`

# code/unix/linux_snd.c
## File Purpose
Linux/FreeBSD platform-specific DMA sound driver for Quake III Arena. It opens the OSS `/dev/dsp` device, configures it for mmap-based DMA audio output, and implements the `SNDDMA_*` interface consumed by the portable sound mixing layer.

## Core Responsibilities
- Register and validate sound CVARs (`sndbits`, `sndspeed`, `sndchannels`, `snddevice`)
- Open the OSS sound device with privilege escalation (`seteuid`)
- Negotiate sample format, rate, and channel count via `ioctl`
- Memory-map the DMA ring buffer into `dma.buffer`
- Arm the DSP trigger to begin output
- Query the current playback pointer (`GETOPTR`) each frame
- Work around a glibc `memset` bug via a custom `Snd_Memset` fallback

## External Dependencies
- **System headers:** `<unistd.h>`, `<fcntl.h>`, `<sys/ioctl.h>`, `<sys/mman.h>`, `<linux/soundcard.h>` (Linux) / `<sys/soundcard.h>` (FreeBSD)
- **Local headers:** `../game/q_shared.h`, `../client/snd_local.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`, global) — `snd_dma.c`
  - `saved_euid` (`uid_t`) — `unix_main.c`
  - `Cvar_Get`, `Com_Printf`, `Com_Memset` — engine core

# code/unix/qasm.h
## File Purpose
A shared header file for x86 assembly (`.s`/`.nasm`) translation units on Unix/Linux, providing C-to-assembly symbol name mangling, architecture detection macros, `.extern` declarations for all software-renderer and audio globals, and byte-offset constants for key C structs used directly from assembly code.

## Core Responsibilities
- Define the `C(label)` macro to handle ELF vs non-ELF symbol name decoration (`_` prefix)
- Detect x86 architecture and set `id386` accordingly
- Declare `.extern` references to all software-renderer globals (z-buffer, texture, lighting, span, edge, surface state) for use in `.s` assembly files
- Declare `.extern` references to audio mixer globals (`paintbuffer`, `snd_p`, etc.)
- Define byte-offset constants for C structs (`plane_t`, `hull_t`, `channel_t`, `edge_t`, `surf_t`, etc.) so assembly can perform field-access without the C type system
- Mirror C struct layouts precisely; comments throughout warn that offsets must stay in sync with their C counterparts

## External Dependencies
- No `#include` directives — entirely self-contained preprocessor/assembler definitions.
- Depends implicitly on the following C headers staying in sync (noted in comments):
  - `model.h` — `plane_t`, `hull_t`, `medge_t`, `mvertex_t`, `mtriangle_t`, `dnode_t`
  - `sound.h` — `sfxcache_t`, `channel_t`, `portable_samplepair_t`
  - `r_shared.h` — `espan_t`, `edge_t`, `surf_t`
  - `d_local.h` — `sspan_t`
  - `d_polyset.c` — `spanpackage_t`
  - `r_local.h` — `clipplane_t`, `NEAR_CLIP`, `CYCLE`
  - `render.h` — `refdef_t`
- External symbols used but defined elsewhere (selected significant ones):

| Symbol | Likely Owner |
|---|---|
| `d_pzbuffer`, `d_zistepu`, `d_ziorigin` | Software renderer depth/z subsystem |
| `paintbuffer`, `snd_p`, `snd_out`, `snd_vol` | Audio mixer (`snd_mix.c`) |
| `r_turb_*` | Turbulent surface rasterizer |
| `edge_p`, `surface_p`, `surfaces`, `span_p` | Renderer edge/surface list manager |
| `aliastransform`, `r_avertexnormals` | Alias model renderer |
| `D_PolysetSetEdgeTable`, `D_RasterizeAliasPolySmooth` | Polyset rasterizer (C entry points called from ASM) |
| `vright`, `vup`, `vpn` | View orientation vectors |

# code/unix/unix_glw.h
## File Purpose
Declares the platform-specific OpenGL window state structure for Linux/FreeBSD. It defines a single shared state object used by the Unix OpenGL window and rendering subsystem.

## Core Responsibilities
- Guards inclusion to Linux/FreeBSD platforms only via a compile-time `#error` directive
- Defines the `glwstate_t` struct holding Unix GL window state
- Exposes `glw_state` as an `extern` global for use across the Unix GL subsystem

## External Dependencies
- `<stdio.h>` — implied by `FILE *log_fp` (must be included before this header by consumers)
- `linux_glimp.c` — defines `glw_state` (definition lives elsewhere)
- No Quake-specific headers; this file is intentionally minimal and low-level

# code/unix/unix_main.c
## File Purpose
This is the Linux/Unix platform entry point for Quake III Arena, implementing the OS-level system layer. It provides the `main()` function, the event loop, DLL loading, TTY console I/O, and all `Sys_*` functions required by the engine's platform abstraction.

## Core Responsibilities
- Houses `main()`: parses args, initializes engine via `Com_Init`, and runs the main `Com_Frame` loop
- Implements the system event queue (`Sys_QueEvent` / `Sys_GetEvent`) feeding input, console, and network events
- Provides TTY console with raw-mode input, line editing, tab completion, and command history
- Implements `Sys_LoadDll` / `Sys_UnloadDll` for native game/cgame/ui module loading via `dlopen`
- Implements `Sys_Error`, `Sys_Quit`, `Sys_Exit` — the unified shutdown/error paths
- Provides no-op or pass-through background file streaming stubs
- Configures architecture cvar and FPU state at startup

## External Dependencies
- **Includes:** `<dlfcn.h>`, `<termios.h>`, `<sys/time.h>`, `<signal.h>`, `<mntent.h>` (Linux), `<fpu_control.h>` (Linux i386)
- **Defined elsewhere:** `Com_Init`, `Com_Frame`, `NET_Init`, `CL_Shutdown`, `IN_Init/Shutdown/Frame`, `Sys_SendKeyEvents`, `Sys_GetPacket`, `Sys_Milliseconds`, `FS_BuildOSPath`, `FS_Read`, `FS_Seek`, `Field_Clear`, `Field_CompleteCommand`, `Z_Malloc`, `Z_Free`, `Cvar_Get`, `Cvar_Set`, `Cvar_VariableString`, `Cmd_AddCommand`, `MSG_Init`, `InitSig`, `Sys_Cwd`, `Sys_SetDefaultCDPath`, `Sys_GetCurrentUser`

# code/unix/unix_net.c
## File Purpose
Implements the Unix/Linux (and macOS) platform-specific network layer for Quake III Arena, providing UDP socket creation, packet send/receive, local address enumeration, and LAN classification. It fulfills the `Sys_*` and `NET_*` network API required by the engine's platform-agnostic common layer (`qcommon`).

## Core Responsibilities
- Convert between engine `netadr_t` and POSIX `sockaddr_in` representations
- Open and close UDP sockets for IP (and stub IPX) communication
- Send and receive raw UDP packets
- Enumerate the host's local IP addresses (platform-divergent: Mac vs. generic POSIX)
- Classify an address as LAN or WAN (RFC 1918 class A/B/C awareness)
- Provide a blocking/sleeping select-based idle for dedicated server frame throttling

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `byte`, `netadr_t`, `cvar_t`, `Com_Printf`, `Com_Error`, `Q_stricmp`, `Com_sprintf`
- `../qcommon/qcommon.h` — `msg_t`, `netadrtype_t`, `NET_AdrToString`, `Cvar_Get`, `Cvar_SetValue`, `PORT_SERVER`, `com_dedicated`
- POSIX headers: `<sys/socket.h>`, `<netinet/in.h>`, `<netdb.h>`, `<arpa/inet.h>`, `<sys/ioctl.h>`, `<errno.h>`
- macOS-only: `<sys/sockio.h>`, `<net/if.h>`, `<net/if_dl.h>`, `<net/if_types.h>`
- **Defined elsewhere:** `NET_AdrToString`, `com_dedicated`, `stdin_active`

# code/unix/unix_shared.c
## File Purpose
Provides Unix/Linux platform-specific system utility functions shared across the engine — timing, filesystem enumeration, path resolution, and miscellaneous CPU/user queries. It implements the `Sys_*` interface declared in `qcommon.h` for POSIX-compliant platforms.

## Core Responsibilities
- High-resolution millisecond timer via `gettimeofday`
- Sub-frame X11 event timing correction (Linux non-dedicated only)
- Directory creation (`Sys_Mkdir`)
- Recursive and filtered file listing (`Sys_ListFiles`, `Sys_ListFilteredFiles`)
- Platform path resolution: CD path, install path, home path
- Current user and processor count queries
- Optional PPC/Apple `Sys_SnapVector` / `fastftol` fallbacks

## External Dependencies
- **Includes:** `<sys/types.h>`, `<sys/stat.h>`, `<errno.h>`, `<stdio.h>`, `<dirent.h>`, `<unistd.h>`, `<sys/mman.h>`, `<sys/time.h>`, `<pwd.h>`
- **Local headers:** `../game/q_shared.h`, `../qcommon/qcommon.h`
- **Defined elsewhere:** `CopyString`, `Z_Malloc`, `Z_Free`, `Com_sprintf`, `Com_FilterPath`, `Q_stricmp`, `Q_strncpyz`, `Q_strcat`, `Sys_Error`, `Com_Printf`; `cvar_t *in_subframe` (declared `extern`, defined in Linux input code)

# code/unix/vm_x86.c
## File Purpose
This is the Linux/Unix x86-specific stub for the Quake III Virtual Machine (Q3VM) JIT compiler. It provides empty placeholder implementations of `VM_Compile` and `VM_CallCompiled`, indicating the x86 JIT backend was not implemented (or not yet ported) for this Unix target.

## Core Responsibilities
- Satisfies the linker requirement for `VM_Compile` and `VM_CallCompiled` on Unix/x86 builds
- Acts as a no-op stub — the Unix build falls back to the interpreted VM path (`VM_CallInterpreted`) rather than JIT-compiled execution
- Mirrors the interface contract declared in `vm_local.h`

## External Dependencies

- **`../qcommon/vm_local.h`** — brings in `vm_t`, `vmHeader_t`, `opcode_t`, `vmSymbol_t`, and the full Q3VM interface declarations
- **`../game/q_shared.h`** (transitively) — base types (`qboolean`, `byte`, `MAX_QPATH`, etc.)
- **`qcommon.h`** (transitively) — `vmHeader_t` definition and common engine declarations
- **Defined elsewhere:** `VM_PrepareInterpreter`, `VM_CallInterpreted`, `currentVM`, `vm_debugLevel` — all implemented in `qcommon/vm_interpreted.c` and `qcommon/vm.c`

