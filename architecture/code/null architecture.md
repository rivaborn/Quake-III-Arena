# Subsystem Overview

## Purpose
The `code/null` subsystem provides stub/no-op implementations of all platform-specific engine interfaces — networking, input, sound, graphics, and system calls — required to compile and link Quake III Arena in headless, dedicated-server, or new-platform-porting configurations. It satisfies the linker's symbol requirements without providing any real functionality, allowing the engine core to run without display, audio, or network hardware.

## Key Files

| File | Role |
|---|---|
| `null_main.c` | Program entry point and stub `Sys_*` platform abstraction layer; drives `Com_Init` / `Com_Frame` loop |
| `null_client.c` | Stub client subsystem; satisfies all `CL_*` symbol requirements for non-client (server-only) builds |
| `null_glimp.c` | Stub `GLimp_*` renderer platform interface and `QGL_*` OpenGL dynamic loader; declares extension function pointers as null |
| `null_input.c` | No-op `IN_Init`, `IN_Frame`, `IN_Shutdown`, and `Sys_SendKeyEvents`; disables all input handling |
| `null_net.c` | Stub network layer; no-op packet send/receive, loopback-only address resolution |
| `mac_net.c` | Mac-specific variant of the null network stub; same loopback-only and no-op packet behavior |
| `null_snddma.c` | No-op `SNDDMA_*` sound DMA driver and `S_*` sound API stubs; signals audio as non-functional |

## Core Responsibilities

- Provide a minimal `main()` entry point that calls `Com_Init` and runs the `Com_Frame` loop, serving as a baseline for porting to new platforms
- Stub all `Sys_*` platform functions (`Sys_SendPacket`, `Sys_GetPacket`, `Sys_SendKeyEvents`, file I/O, error reporting) so `qcommon` can link without a real platform driver
- Satisfy the full client (`CL_*`) symbol table for dedicated-server builds, with a minimal `CL_Init` that registers `cl_shownet`
- Declare all OpenGL extension function pointers required by the renderer and provide no-op `GLimp_*` / `QGL_*` lifecycle hooks
- Resolve only `"localhost"` to `NA_LOOPBACK` in `NET_StringToAdr`; silently discard all outbound packets and report no inbound packets
- Allow `SNDDMA_*` and `S_*` sound API calls to complete without error, returning `qfalse`/`0` to signal no audio hardware
- Enable the entire engine codebase to compile and link in the absence of any real OS, display, audio, or network backend

## Key Interfaces & Data Flow

**Exposes to other subsystems:**
- `Sys_*` platform abstraction symbols consumed by `code/qcommon/common.c` and related qcommon modules
- `CL_*` client API symbols consumed by the server and qcommon link units in dedicated builds
- `GLimp_*` and `QGL_*` symbols consumed by `code/renderer/tr_init.c` and the renderer subsystem
- `IN_*` and `Sys_SendKeyEvents` consumed by the client input dispatch path
- `SNDDMA_*` and `S_*` consumed by the portable sound mixer in `code/client/snd_dma.c`
- `NET_StringToAdr`, `Sys_SendPacket`, `Sys_GetPacket` consumed by `code/qcommon/net_chan.c` and the networking layer

**Consumes from other subsystems:**
- `../qcommon/qcommon.h` — `Com_Init`, `Com_Frame`, `netadr_t`, `msg_t`, `NA_LOOPBACK`, and all `Sys_*` / `NET_*` declarations
- `../client/client.h` — transitively pulls in `q_shared.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- `../renderer/tr_local.h` — OpenGL types (`GLenum`, `GLuint`), `GLimp_*` / `QGL_*` declarations
- `Cvar_Get` from `code/qcommon/cvar.c` (used only in `null_client.c` to register `cl_shownet`)
- Standard C library: `<stdio.h>`, `<errno.h>`, `strcmp`, `memset`

## Runtime Role

- **Init:** `null_main.c`'s `main()` calls `Com_Init`, which in turn triggers `CL_Init` (stubbed in `null_client.c`), renderer init (stubbed `GLimp_Init`/`QGL_Init` in `null_glimp.c`), input init (`IN_Init` no-op), and sound init (`SNDDMA_Init` returning `qfalse`)
- **Frame:** `Com_Frame` is called in a loop; `IN_Frame` and `Sys_SendKeyEvents` execute as no-ops; network send/receive stubs are called and return immediately; no rendering or audio output occurs
- **Shutdown:** `GLimp_Shutdown`, `IN_Shutdown`, and `QGL_Shutdown` are called as no-ops; the process exits via `Sys_Quit` (stubbed) or on fatal error via `Sys_Error` printing to stdout

## Notable Implementation Details

- `mac_net.c` and `null_net.c` are parallel files with identical behavior; `mac_net.c` appears to be a Mac-platform-targeted copy of the null network stub rather than a distinct implementation
- `null_glimp.c` is responsible for **declaring** (not merely stubbing) the OpenGL extension function pointers (WGL/ARB/EXT); in real platform builds these are declared in the platform-specific glimp files instead
- The subsystem is explicitly documented in `null_main.c` as a **porting aid**, making it the canonical starting point for bringing up the engine on a new, unsupported platform
- All boolean-returning stubs consistently return `qfalse` for init/capability queries, which signals to callers that the relevant subsystem is unavailable rather than silently succeeding
