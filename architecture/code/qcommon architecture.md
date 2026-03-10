# Subsystem Overview

## Purpose
`qcommon` is the shared engine core of Quake III Arena, providing all subsystem services required by both the client and server: collision detection, command/cvar execution, virtual filesystem, network messaging and channel management, virtual machine hosting, and foundational memory and error infrastructure. It acts as the integration layer between platform-specific code (`win32/`, `unix/`) and the game-facing modules (`cgame`, `game`, `ui`).

## Key Files

| File | Role |
|---|---|
| `common.c` | Engine init/shutdown, per-frame loop, zone/hunk allocators, error handling, event queue |
| `qcommon.h` | Central shared header; aggregates all subsystem API declarations for engine-level consumers |
| `qfiles.h` | On-disk binary format definitions (BSP, MD3, MD4, QVM, PCX/TGA); shared with tool utilities |
| `cmd.c` | Command buffer (FIFO), command tokenization, handler registration and dispatch |
| `cvar.c` | Console variable system: registration, protection flags, VM bridging, config serialization |
| `files.c` | Virtual filesystem: search path hierarchy, `.pk3` (ZIP) archive indexing, pure-server enforcement |
| `cm_load.c` | BSP map loader: deserializes all lumps into runtime `clipMap_t cm`, initializes box hull and area flood |
| `cm_local.h` | Private CM header: all internal collision types (`clipMap_t`, `cbrush_t`, `cLeaf_t`, `traceWork_t`, etc.) |
| `cm_trace.c` | AABB/capsule swept-volume trace and position-test through BSP and patch surfaces |
| `cm_test.c` | BSP point-in-leaf queries, content flags, PVS cluster data, area portal flood-connectivity |
| `cm_patch.c` | Bezier patch mesh collision: subdivision, facet/plane generation, trace and position tests |
| `cm_patch.h` | Patch collision types (`patchCollide_t`, `facet_t`, `patchPlane_t`, `cGrid_t`) and entry point declarations |
| `cm_public.h` | Public CM API: map load/unload, clip handles, trace functions, PVS and portal queries |
| `cm_polylib.c` | Winding (polygon) geometry utilities for CM debug/visualization only; not in the runtime trace path |
| `cm_polylib.h` | `winding_t` type and polygon operation declarations |
| `msg.c` | Network message serialization: bit-level I/O, Huffman-compressed bitstream, delta-compression for `usercmd_t`/`entityState_t`/`playerState_t` |
| `net_chan.c` | Reliable sequenced UDP channel (`netchan_t`): fragmentation/reassembly, loopback, OOB datagrams, address utilities |
| `huffman.c` | Adaptive Huffman codec for network message compression |
| `md4.c` | MD4 hash; exposes `Com_BlockChecksum` and `Com_BlockChecksumKey` for data integrity |
| `vm.c` | VM lifecycle: load, restart, free; `VM_Call` dispatch to DLL/JIT/interpreter; symbol table and profiling |
| `vm_local.h` | Internal VM types: full QVM opcode set, `vm_t` runtime state, `vmSymbol_t`, backend prototypes |
| `vm_interpreted.c` | Software interpreter backend: bytecode preparation, fetch-decode-execute loop, sandbox enforcement |
| `vm_x86.c` | x86 JIT compiler backend: two-pass Q3VM→x86 translation, `AsmCall` trampoline, `VM_CallCompiled` |
| `vm_ppc.c` | PowerPC JIT compiler backend (original): three-pass Q3VM→PPC translation, `AsmCall`, `VM_CallCompiled` |
| `vm_ppc_new.c` | PowerPC JIT compiler backend (revised): register-tracked operand stack, float/int patch-back, `AsmCall` |
| `unzip.c` | Embedded ZIP decompression library (zlib/minizip); provides `unzFile` API to `files.c` |
| `unzip.h` | Public `unzFile` API and internal ZIP streaming state declarations |
| `huffman.c` | Adaptive Huffman codec used by `msg.c` and `net_chan.c` |

## Core Responsibilities

- **Foundation services:** Zone and hunk memory allocation, `longjmp`-based error recovery (`Com_Error`), per-frame event dispatch, and engine init/shutdown sequencing (`common.c`).
- **Configuration:** Console variable lifecycle management with protection flags (`CVAR_ROM`, `CVAR_LATCH`, `CVAR_CHEAT`), VM bridging via `vmCvar_t`, and config-file serialization (`cvar.c`).
- **Command execution:** Buffered text-based command FIFO, tokenization, registered handler dispatch, and forwarding to cgame/game/UI/server (`cmd.c`).
- **Virtual filesystem:** Transparent merging of directory trees and `.pk3` ZIP archives with priority ordering, pure-server pak validation, and demo/restricted-mode enforcement (`files.c`, `unzip.c`).
- **Collision model:** Full BSP collision world: map load, AABB/capsule sweep traces, point content tests, PVS cluster queries, area portal connectivity, and Bezier patch collision (`cm_load.c`, `cm_trace.c`, `cm_test.c`, `cm_patch.c`).
- **Network messaging:** Bit-level serialization, Huffman-compressed bitstreams, delta-compression of game-state structures, reliable sequenced UDP channels with fragmentation/reassembly (`msg.c`, `net_chan.c`, `huffman.c`).
- **Virtual machine hosting:** Lifecycle management for up to three QVM instances (cgame, game, ui); routing calls to native DLL, x86/PPC JIT, or software interpreter; sandbox enforcement and developer symbol/profile support (`vm.c`, `vm_interpreted.c`, `vm_x86.c`, `vm_ppc.c`, `vm_ppc_new.c`).
- **Data integrity:** MD4 checksums for map verification, pak validation, and CD-key schemes (`md4.c`).

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `cm_public.h` — the entire CM public API (`CM_LoadMap`, `CM_BoxTrace`, `CM_PointContents`, `CM_ClusterPVS`, `CM_AdjustAreaPortalState`, etc.) consumed by the server (`sv_world.c`, `sv_init.c`), client (`cl_cgame.c`), and game VM.
- `qcommon.h` — aggregated declarations for `Cmd_*`, `Cvar_*`, `FS_*`, `MSG_*`, `Netchan_*`, `VM_*`, `Com_*`, `Z_*`, `Hunk_*`, `Huff_*`, `Sys_*` consumed by all engine-level translation units.
- `VM_Call` — single entry point for all cgame/game/ui module invocations; hides DLL/JIT/interpreter selection.
- `Netchan_Transmit` / `Netchan_Process` — reliable packet I/O consumed by `cl_net_chan.c` and `sv_net_chan.c`.

**Consumed from other subsystems:**
- Platform layer (`win32/`, `unix/`) — `Sys_SendPacket`, `Sys_LoadDll`, `Sys_ListFiles`, `Sys_Mkdir`, `Sys_*Path`, and low-level I/O.
- Client (`CL_*`) and server (`SV_*`) frame-tick entry points called from `Com_Frame` in `common.c`.
- Game-shared math (`q_shared.c`) — `VectorCopy`, `DotProduct`, `CrossProduct`, `AngleVectors`, `BoxOnPlaneSide`, `PlaneTypeForNormal`, `SetPlaneSignbits` used throughout CM and VM code.
- `BotDrawDebugPolygons` — called from `cm_patch.c` for debug visualization; defined in the bot library.

## Runtime Role

- **Init (`Com_Init`):** `common.c` initializes the zone/hunk allocators, registers core cvars, then sequentially brings up `Cmd`, `Cvar`, `FS` (filesystem), `NET`, `Netchan`, VM infrastructure, and finally calls `CL_Init`/`SV_Init`. The CM subsystem is initialized lazily on the first `CM_LoadMap` call (triggered by `SV_SpawnServer`).
- **Frame (`Com_Frame`):** Dispatches queued input/network events, calls `SV_Frame` (server tick) and `CL_Frame` (client tick). The CM subsystem is purely on-demand during the frame: trace queries are issued by `sv_world.c` and game VM syscalls; no CM per-frame tick exists. `Cbuf_Execute` drains the command buffer each frame before the server/client ticks.
- **Shutdown (`Com_Shutdown`):** Tears down VM instances (`VM_Free`), filesystem search paths, network channels, and zone/hunk memory. `CM_ClearMap` zeroes the global `cm` and frees hunk data for the next map load.

## Notable Implementation Details

- **Hunk allocator is dual-ended:** `common.c` maintains a single contiguous hunk buffer used from both the low end (permanent data) and high end (temporary/overwritten data), with a `Mark`/`ClearToMark` mechanism for transient high allocations. The CM lump data and QVM code images are allocated from the low (permanent) end.
- **CM patch collision is precomputed:** `cm_patch.c` subdivides Bezier control grids at load time into a flat `patchCollide_t` (facets + planes), adding axial and edge bevel planes to prevent tunneling. Runtime traces (`cm_trace.c`) operate on this precomputed structure with no further tessellation.
- **VM has three execution backends selected per-platform:** `vm.c` selects DLL (fastest, not shipped in pure mode), JIT-compiled (`vm_x86.c` on x86, `vm_ppc.c`/`vm_ppc_new.c` on PPC), or software interpreter (`vm_interpreted.c`) based on `vmInterpret_t`. All three share the same `VM_Call` dispatch point.
- **QVM pointer sandboxing:** `vm_interpreted.c` and the JIT backends enforce a `dataMask` on all VM memory accesses, preventing bytecode from reading or writing outside the VM's own data segment.
- **Adaptive Huffman codec is stateful and symbol-driven:** `huffman.c` updates the tree after each symbol transmitted or received; both encoder and decoder maintain identical tree state by applying the same update sequence, so no separate code table is transmitted.
- **`unzip.c` replaces zlib's allocator:** `zcalloc`/`zcfree` are remapped to Q3's `Z_Malloc`/`Z_Free` so ZIP decompression participates in the engine's tracked zone memory rather than using `malloc` directly.
- **Command dispatch order is fixed:** `cmd.c` tries handlers in the sequence: registered `xcommand_t` → cvar set → cgame → game → UI → server forward; only one handler fires per command invocation.
- **`cm_polylib.c` is diagnostic-only:** The winding utilities it provides are used exclusively by debug visualization paths (e.g., `CM_DrawDebugSurface`) and are not part of any runtime collision or trace computation.
