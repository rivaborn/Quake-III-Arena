# code/qcommon/common.c — Enhanced Analysis

## Architectural Role

`common.c` is the engine's root orchestrator: the single file responsible for bootstrapping every subsystem in dependency order and driving the main loop that coordinates them each frame. It sits at the top of the `qcommon` layer, which the architecture context identifies as the "integration layer and service backbone" — and `common.c` is the apex of that layer. All major subsystems (Renderer via `ri`, Client, Server, botlib, VM host, FS, Netchan) are either initialized here or receive services (memory, error, printing, events) defined here. Nothing in the codebase is architecturally "above" this file at runtime.

The zone and hunk allocators defined here are foundational infrastructure: virtually every subsystem — renderer (`ri.Hunk_Alloc`), botlib (`botimport` vtable memory calls), VM host, filesystem — ultimately routes its persistent allocations through this file's pools. This makes `common.c` both the frame orchestrator and the memory foundation for the entire engine session.

## Key Cross-References

### Incoming (who depends on this file)

- **Every subsystem** calls `Com_Printf` / `Com_DPrintf` / `Com_Error` — these are truly engine-global APIs; the cross-reference index shows hundreds of call sites across `client/`, `server/`, `botlib/`, `renderer/`, `game/`, and `qcommon/`.
- **Renderer** accesses hunk through `refimport_t ri`: `ri.Hunk_Alloc` maps directly to `Hunk_Alloc` here; renderer has no direct zone access.
- **botlib** accesses memory through `botlib_import_t botimport` vtable, which server bridges to zone/hunk calls from this file.
- **VM system** (`vm.c`) calls `Hunk_Alloc` for QVM image loading and `Z_Malloc` for VM descriptor structs.
- **Filesystem** (`files.c`) calls `Hunk_Alloc` for pack file headers and `Z_Malloc` for dynamic file state.
- **Server** (`sv_main.c`, `sv_snapshot.c`) reads `com_frameTime`, `com_sv_running`, and calls into `Com_EventLoop` indirectly via the frame dispatch.
- **Client** (`cl_main.c`) reads `com_cl_running`, `com_timescale`, `com_fixedtime`, `com_maxfps`; `cl_paused` is defined here and read in `CL_Frame`.
- `abortframe` (`jmp_buf`) is set here and is the target of `longjmp` called from `Com_Error`, which is reached from any subsystem on ERR_DROP.
- `com_fullyInitialized` is read by `Com_WriteConfig_f` to guard premature config serialization.

### Outgoing (what this file depends on)

- **All CL_\* entry points** (`CL_Init`, `CL_Frame`, `CL_Shutdown`, `CL_Disconnect`, `CL_FlushMemory`, `CL_KeyEvent`, `CL_MouseEvent`, `CL_PacketEvent`, `CL_ConsolePrint`, `CL_CDDialog`, `CL_StartHunkUsers`) — client layer is driven from here.
- **All SV_\* entry points** (`SV_Init`, `SV_Frame`, `SV_Shutdown`, `SV_PacketEvent`) — server layer is driven from here.
- **FS_\*** for filesystem init, file I/O (log file), pure server validation, and initialized-state checks.
- **Cvar_\*, Cmd_\*, Cbuf_\*** for all config/command infrastructure.
- **NET_\*, Netchan_Init, NET_GetLoopPacket** for network initialization and loopback packet polling.
- **VM_Init** for QVM host initialization.
- **Sys_\*** (`Sys_Init`, `Sys_Milliseconds`, `Sys_Print`, `Sys_Error`, `Sys_Quit`, `Sys_GetEvent`) — platform layer abstraction consumed here.
- `Key_WriteBindings`, `UI_usesUniqueCDKey`, `CIN_CloseAllVideos` — UI/input utilities called at config-write time.

## Design Patterns & Rationale

**Dual-ended hunk allocator** (low/high split with temp overlay): This is a classic id Software pattern dating to Quake 1. The low end holds permanent level data (BSP, entities, shaders); the high end holds renderer resources allocated during `CL_StartHunkUsers`. The temp region overlays the current permanent end and is reset each frame. This design makes map load/unload trivially cheap (reset the high-water mark) at the cost of strict allocation discipline. Modern engines use similar frame-allocator / scratch-buffer patterns.

**Zone allocator with rover and tag-based mass free**: First-fit scan with a roving start pointer reduces average scan length. Tags enable `Z_FreeTags` to bulk-free all allocations from a subsystem (e.g., clearing all UI assets without enumerating them). The `smallzone` split prevents small allocations from fragmenting the main pool — an early form of size-class separation.

**`setjmp`/`longjmp` for recoverable errors**: Pre-C++ exception handling. `Com_Init` and `Com_Frame` each establish a `setjmp` checkpoint; `Com_Error(ERR_DROP)` unwinds via `longjmp` directly to the frame boundary, bypassing the call stack. This is safe only because the hunk and zone allocators survive the jump (they're reset explicitly, not via destructors). The escalation to `ERR_FATAL` on rapid repeated drops is a watchdog against infinite error loops.

**Output redirection (rd_buffer/rd_flush)**: The `Com_BeginRedirect`/`Com_EndRedirect` pair allows `Com_Printf` output to be buffered into a caller-supplied buffer and flushed via a callback. This is the mechanism for RCON: the server temporarily redirects all print output to a network packet buffer, executes the command, then flushes it to the remote client.

**Journal replay**: The event queue can be journaled to disk and replayed deterministically. This is a developer tool for reproducing crashes — a form of input recording before it became a mainstream engine feature.

## Data Flow Through This File

```
Platform (Sys_GetEvent)
    │
    ▼
com_pushedEvents ring buffer (push via Com_QueueEvent)
    │
    ▼
Com_EventLoop ──► key/mouse ──► CL_KeyEvent / CL_MouseEvent
              ──► packets   ──► SV_PacketEvent / CL_PacketEvent / NET_GetLoopPacket
              ──► console   ──► Cbuf_AddText
    │
    ▼
Cbuf_Execute (flush command FIFO)
    │
    ▼
Com_Frame timing ──► SV_Frame(msec) ──► [game VM, bot AI, snapshot]
                 └──► CL_Frame(msec) ──► [cgame VM, renderer, audio]
```

Memory allocation is a parallel static flow:
- `Z_Malloc` / `Z_Free` → `mainzone` doubly-linked list (persists across map loads)
- `S_Malloc` → `smallzone` (same lifetime, separate pool)
- `Hunk_Alloc` → `s_hunkData[0..hunk_low.permanent]` or `[hunk_high.permanent..s_hunkTotal]` (cleared on map change)
- `Hunk_AllocateTempMemory` → overlay at current permanent mark; LIFO discipline enforced by sentinel check

## Learning Notes

- **The hunk is the precursor to modern frame/scratch allocators**. The "temp memory" concept — allocate on top of permanent, reset to mark when done — is structurally identical to frame allocators used in AAA engines today, just without a formal API name.
- **`QDECL` is a calling convention macro** (`__cdecl` on Win32, empty on others). It appears on `Com_Printf`, `Com_Error`, etc. because these may be called through function pointers stored in vtables (`refimport_t`, `botlib_import_t`) where the calling convention must match exactly across DLL boundaries.
- **`com_speeds` and the timing globals** (`time_game`, `time_frontend`, `time_backend`) represent an early performance profiling system built directly into the frame loop — the ancestor of GPU/CPU frame timing overlays in modern engines.
- **The "safe mode" startup** (detecting `+safe` or `+cvar_restart` on the command line to skip config loading) is a resilience pattern that persists in modern games: a way for users to recover from a bad config without a reinstall.
- **There is no scene graph, ECS, or component system here**. The engine is purely procedural: a frame loop that calls into subsystems sequentially. Entity behavior is encoded in think functions in `g_main.c`, not composed from components.

## Potential Issues

- **`vsprintf` without length limit in `Com_Error`** (line ~270): Uses `vsprintf` into `com_errorMessage[MAXPRINTMSG]`, unlike `Com_Printf` which uses the safe `Q_vsnprintf`. A sufficiently long format expansion could silently overflow the buffer — particularly dangerous since this runs during an error path when state is already compromised.
- **`opening_qconsole` static flag in `Com_Printf` is not thread-safe**: If SMP rendering is active (front-end and back-end on separate threads, both calling `Com_Printf`), the flag can race. In practice the renderer's `ri.Printf` calls are infrequent during init, but this is a latent hazard.
- **ERR_DROP burst escalation uses static state with no reset**: `errorCount` and `lastErrorTime` are static locals in `Com_Error`. If three drops occur in 100 ms legitimately (e.g., during map loading under load), they escalate to `ERR_FATAL` — an overly aggressive heuristic that can mask the root cause.
