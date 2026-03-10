# code/cgame/cg_syscalls.c — Enhanced Analysis

## Architectural Role
This file implements the **cgame DLL's only interface to the engine kernel**, serving as an ABI adapter that translates typed C function calls into integer-dispatched syscalls. It is the sandbox boundary: cgame has zero direct access to engine internals and must funnel all requests through this dispatcher, making this file essential to cgame's process isolation and DLL reloadability. The opcodes dispatched here (`CG_*`) are the contract between cgame and the engine at runtime—any mismatch causes silent memory corruption or crashes.

## Key Cross-References

### Incoming (who depends on this file)
- **Every cgame module**: `cg_main.c`, `cg_view.c`, `cg_ents.c`, `cg_draw.c`, `cg_predict.c`, `cg_snapshot.c`, `cg_event.c`, `cg_players.c`, `cg_weapons.c`, `cg_marks.c`, `cg_particles.c`, `cg_servercmds.c`, `cg_consolecmds.c`, etc. — all call `trap_*` functions from here
- **Engine initialization**: `client/cl_cgame.c:CL_CgameSystemCalls()` calls `dllEntry` immediately after loading the DLL via `Sys_LoadDll`
- **No other file directly references this file** because cgame is monolithic (no inter-cgame module syscalling)

### Outgoing (what this file depends on)
- **Engine dispatcher** (passed via `dllEntry`): routed through the `syscall` function pointer
  - Ultimately dispatches into `qcommon/vm.c:VM_Call()` or equivalent handler, which routes to:
    - `qcommon` subsystem: `cmd.c`, `cvar.c`, `files.c`, `cm_*.c`, `msg.c`, `net_chan.c` (via `qcommon.h` opcodes)
    - `renderer` subsystem: `tr_*.c` family (via `tr_public.h` opcodes)
    - `client` subsystem: input, snapshot, console (via internal client handlers)
    - `sound` subsystem: `snd_dma.c`, `snd_mix.c` (via `snd_public.h` opcodes)
- **Opcode definitions** from `cg_public.h` (e.g., `CG_PRINT`, `CG_R_RENDERSCENE`) — shared header between engine and cgame
- **Type definitions**: transitively via `cg_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `qcommon.h`

## Design Patterns & Rationale

**Adapter / Facade Pattern**: Wraps an untyped, integer-only dispatcher (`syscall`) with typed C functions (`trap_*`), hiding the bit-level calling convention.

**Type-Punning for ABI Compatibility**: `PASSFLOAT()` converts floats to their raw int bit-pattern because the syscall variadic ABI (inherited from QVM architecture) treats all arguments as `int`. This avoids float-to-int conversion loss and matches the QVM assembler's handling of floats.

**Sandbox via Function Pointer Indirection**: The `syscall` pointer is passed in from the engine (not linked at compile time). This allows:
- cgame to be reloaded without relinking the engine
- Complete isolation—cgame cannot access engine code except via syscall
- Versioning: a new engine can provide a new syscall function with different opcodes, preventing stale-cgame crashes (in theory; in practice the opcodes are fixed)

**QVM Compatibility Layer**: Excluded from QVM builds (`#ifdef Q3_VM #error`). The QVM build uses `cg_syscalls.asm` (pure assembler), which maps the same `trap_*` symbols to Q3VM inline syscall instructions. Keeps source-level API identical across both execution models.

**Subsystem Demultiplexing**: One dispatcher (`syscall`) routes to all engine subsystems via opcode. This is simpler than multiple function pointers but couples all subsystems through a single ABI boundary.

## Data Flow Through This File

1. **Initialization Phase**:
   - Engine calls `dllEntry(syscallptr)` → cgame receives engine's dispatcher
   - `syscall` pointer now points to live dispatcher (was sentinel `-1`)

2. **Per-Frame Flow** (repeats from `CG_DrawActiveFrame` onward):
   - cgame calls `trap_Snapshot()` → `syscall(CG_GETSNAPSHOT, ...) ` → engine returns server snapshot
   - cgame calls `trap_GetUserCmd()` → `syscall(CG_GETUSERCMD, ...) ` → engine returns client's input
   - cgame calls `trap_CM_BoxTrace()` → `syscall(CG_CM_BOXTRACE, ...) ` → engine collision subsystem
   - cgame calls `trap_R_*()` (e.g., `trap_R_AddRefEntityToScene()`) → renderer back-end accumulates scene
   - cgame calls `trap_S_*()` (e.g., `trap_S_StartSound()`) → sound mixer queues audio
   - cgame calls `trap_R_RenderScene()` → `trap_UpdateScreen()` → engine flushes frame to display

3. **Argument Marshalling**:
   - Pointers (e.g., `trace_t *results`, `refEntity_t *re`) are passed as-is through variadic dispatcher
   - Floats (e.g., `intensity` in `trap_R_AddLightToScene()`) are converted via `PASSFLOAT()` to avoid conversion
   - Handles/ints (e.g., `sfxHandle_t`, `qhandle_t`) pass through natively

4. **Return Values**:
   - Syscall returns int; some wrappers cast to typed return (`qboolean`, `qhandle_t`, etc.)
   - Pointer-output arguments (e.g., `glconfig_t *glconfig`) are filled by engine in-place

## Learning Notes

**QVM Legacy**: The integer-only syscall ABI stems from the Q3VM bytecode architecture (pre-JIT), which had no native float support. The `PASSFLOAT()` trick became idiomatic across id's engines for decades. Modern engines would use a proper calling convention (stdcall, fastcall, or Rust FFI).

**Opcode Versioning**: Notice the opcodes are bare integers (e.g., `CG_PRINT = 0`, `CG_MILLISECONDS = 4`). If the engine adds a new opcode in the middle, all downstream opcodes shift—cgame *must* rebuild. There is no forward compatibility; cgame and engine are always compiled together.

**No Dynamic Dispatch**: Unlike a vtable-based plugin API, cgame has zero flexibility in which engine functions it calls. The opcode set is hard-coded at compile time. This simplicity comes at the cost of version coupling.

**Collision and Rendering as First-Class Syscalls**: Notice `trap_CM_*()` and `trap_R_*()` are syscalls, not linked libraries. This means collision and rendering can be hot-swapped independently of cgame—e.g., a different renderer DLL can replace the entire OpenGL implementation without touching cgame.

**Sound Mixing is Engine-Side**: `trap_S_*()` syscalls queue commands to the engine's audio mixer. cgame doesn't touch audio buffers directly; it's pure command dispatch.

## Potential Issues

1. **Float Bit-Reinterpretation Assumes IEEE-754**: `PASSFLOAT()` relies on IEEE-754 float layout. On exotic architectures (older PPC) this could cause silent bugs. No runtime checks.

2. **Opcode Collision Risk**: If `cg_public.h` and the engine's opcode dispatch table drift (e.g., due to a merge conflict or partial rebuild), cgame will call the wrong engine functions. No version number or checksum validation.

3. **Pointer Lifetime Across Syscall Boundary**: Some `trap_*` functions (e.g., `trap_Argv()`) fill caller-owned buffers. If cgame passes a stack-allocated buffer and the engine somehow schedules async work, use-after-free is possible (unlikely in this engine's synchronous design, but not impossible in modded variants).

4. **Silent Failure on Uninitialized Syscall**: If `dllEntry` is never called (e.g., corrupted load sequence), `syscall` remains the sentinel `-1` cast as a pointer. Calling any `trap_*()` dereferences this poison value, causing an immediate crash rather than a graceful error message.

5. **No Error Propagation Path**: Many `trap_*()` functions return void and have no way to signal engine errors back to cgame (e.g., `trap_FS_FOpenFile` returns size, but what if the open failed?). cgame must rely on implicit contract (e.g., file size ≤ 0 means error).
