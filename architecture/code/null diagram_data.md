# code/null/mac_net.c
## File Purpose
A null/stub implementation of the Mac network layer for Quake III Arena. It provides non-functional placeholder implementations of the platform-specific network functions required by the engine, always returning failure or doing nothing.

## Core Responsibilities
- Provide a stub `NET_StringToAdr` that only resolves "localhost" to `NA_LOOPBACK`
- Provide a no-op `Sys_SendPacket` that discards all outgoing packet data
- Provide a stub `Sys_GetPacket` that always reports no incoming packets

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `netadr_t`, `netadrtype_t` (`NA_LOOPBACK`), `memset`
- `../qcommon/qcommon.h` — `msg_t`, `netadr_t`, `Sys_SendPacket`/`Sys_GetPacket` declarations
- `strcmp`, `memset` — C standard library (via `q_shared.h` includes)

# code/null/null_client.c
## File Purpose
Provides a null (stub) implementation of the client subsystem for use in dedicated server or headless builds where no actual client functionality is required. All functions have empty bodies or return safe default values.

## Core Responsibilities
- Satisfy the linker's demand for client API symbols in non-client builds
- Provide a no-op `CL_Init` that registers the `cl_shownet` cvar (minimum viable init)
- Return safe defaults (`qfalse`/`qtrue`) from boolean-returning stubs
- Allow the server-side codebase to compile and link without the full client module

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- `Cvar_Get` — defined in `code/qcommon/cvar.c`
- `cvar_t`, `netadr_t`, `msg_t`, `fileHandle_t`, `qboolean` — defined elsewhere in qcommon/game headers

# code/null/null_glimp.c
## File Purpose
This is a null/stub implementation of the platform-specific OpenGL import layer (`GLimp`) and the dynamic OpenGL function pointer loader (`QGL`) for Quake III Arena. It provides empty no-op bodies for all required renderer platform interface functions, intended for headless/server builds or porting scaffolding where no actual display is needed.

## Core Responsibilities
- Declare the OpenGL extension function pointers required by the renderer (WGL/ARB/EXT)
- Provide a no-op `GLimp_EndFrame` so the renderer can call buffer swap without crashing
- Provide a no-op `GLimp_Init` / `GLimp_Shutdown` for renderer lifecycle hooks
- Provide no-op `GLimp_EnableLogging` and `GLimp_LogComment` for debug logging stubs
- Provide a trivially succeeding `QGL_Init` / `QGL_Shutdown` for the OpenGL dynamic loader interface

## External Dependencies
- **`../renderer/tr_local.h`** — pulls in `qboolean`, `qtrue`, OpenGL types (`GLenum`), and the `GLimp_*` / `QGL_*` declarations that this file implements
- `GLenum`, `GLuint`, etc. — defined via OpenGL headers transitively included through `tr_local.h` → `qgl.h`
- All `GLimp_*` and `QGL_*` symbols are **declared in `tr_local.h`** and **defined here** as stubs; the real implementations live in `code/win32/win_glimp.c`, `code/unix/linux_glimp.c`, etc.

# code/null/null_input.c
## File Purpose
Provides a no-op (null) implementation of the platform input subsystem for Quake III Arena. All functions are empty stubs, used when building a headless/dedicated server or a platform-agnostic null client where no actual input handling is needed.

## Core Responsibilities
- Stub out `IN_Init` so the engine's input initialization path can be called safely with no effect
- Stub out `IN_Frame` so the per-frame input polling path executes without error
- Stub out `IN_Shutdown` so the input teardown path completes cleanly
- Stub out `Sys_SendKeyEvents` so the OS key-event pump is a no-op

## External Dependencies
- `../client/client.h` — pulls in the full client subsystem header (key types, `clientActive_t`, `kbutton_t`, input function declarations, etc.), though none of those symbols are actually used here.

**Defined elsewhere (symbols the real implementation would use):**
- `Key_Event` / `Com_QueueEvent` — engine key/event queue (defined in `cl_keys.c` / `common.c`)
- `cl.mouseDx`, `cl.mouseDy` — mouse delta accumulators in `clientActive_t` (defined in `cl_main.c`)
- Platform OS handles — not applicable in null build

# code/null/null_main.c
## File Purpose
A minimal null/stub system driver for Quake III Arena, intended to aid porting efforts to new platforms. It provides no-op or trivially forwarding implementations of all required `Sys_*` platform abstraction functions, and contains the program entry point.

## Core Responsibilities
- Provide a compilable stub for all `Sys_*` platform interface functions required by `qcommon`
- Implement the program entry point (`main`) that initializes the engine and runs the main loop
- Forward streamed file I/O to standard C `fread`/`fseek`
- Print fatal errors to stdout and terminate the process
- Serve as a minimal baseline for porting to platforms without a real system driver

## External Dependencies
- `<errno.h>`, `<stdio.h>` — standard C I/O and error codes
- `../qcommon/qcommon.h` — engine-wide common declarations; defines `Com_Init`, `Com_Frame`, and the full `Sys_*` interface contract
- **Defined elsewhere:** `Com_Init`, `Com_Frame` (in `qcommon/common.c`); all `Sys_*` signatures are declared in `qcommon.h` but the authoritative platform implementations live in `code/win32/`, `code/unix/`, `code/macosx/`

# code/null/null_net.c
## File Purpose
Provides a null (stub) implementation of the platform-specific networking layer for Quake III Arena. It is used in headless or minimal build configurations where real network I/O is not needed, implementing only loopback address resolution.

## Core Responsibilities
- Stub out `Sys_SendPacket` so packet transmission is a no-op
- Stub out `Sys_GetPacket` so packet reception always returns nothing
- Implement `NET_StringToAdr` with minimal support: only resolves `"localhost"` to `NA_LOOPBACK`; all other addresses fail

## External Dependencies
- `../qcommon/qcommon.h` — provides `netadr_t`, `netadrtype_t`, `msg_t`, `qboolean`, `NA_LOOPBACK`, and the declared signatures for `NET_StringToAdr`, `Sys_SendPacket`, and `Sys_GetPacket`
- `strcmp`, `memset` — C standard library (via `qcommon.h` transitively including `q_shared.h`)
- `Sys_SendPacket`, `Sys_GetPacket`, `NET_StringToAdr` — declared in `qcommon.h`; **defined here** as null implementations

# code/null/null_snddma.c
## File Purpose
Provides a null (no-op) implementation of the platform-specific sound DMA driver interface. It exists to allow Quake III Arena to compile and run without any audio hardware or audio subsystem, returning safe default values for all sound queries.

## Core Responsibilities
- Stub out `SNDDMA_*` lifecycle functions so the portable sound mixer has valid symbols to call
- Stub out higher-level `S_*` sound API functions to prevent crashes in headless or null-platform builds
- Return `qfalse`/`0` from all init/query functions to signal audio is non-functional

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h` (for `qboolean`, `sfxHandle_t`) and `snd_public.h` (for the sound API contract)
- `sfxHandle_t` — typedef defined in `snd_public.h` (defined elsewhere)
- `qboolean`, `qfalse` — defined in `q_shared.h` (defined elsewhere)

