# code/qcommon/vm.c — Enhanced Analysis

## Architectural Role

`vm.c` is the **security and isolation boundary** of the entire engine. It defines the contract between the trusted engine core and the three untrusted game modules (cgame, game, ui), each of which may be user-supplied `.qvm` bytecode or native DLLs. All engine-to-VM and VM-to-engine calls funnel through this file's dispatch layer. Within the qcommon subsystem, `vm.c` is a peer of the collision, filesystem, and network modules, but uniquely it is the only module that changes execution context — setting `currentVM` before each call and restoring it after, making it the runtime equivalent of a privilege gate.

The three VMs loaded here correspond exactly to the three user-extensible subsystems described in the architecture overview: cgame (client-side game logic), game (server-side simulation), and ui (menu system). No fourth VM is ever needed; `MAX_VM = 3` is not arbitrary.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/client/cl_cgame.c`** — calls `VM_Create` with `CL_CgameSystemCalls` as the syscall handler to instantiate the cgame VM; calls `VM_Call(cgvm, CG_DRAW_ACTIVE_FRAME, ...)` each rendered frame; calls `VM_Free` on disconnect.
- **`code/client/cl_ui.c`** — same lifecycle pattern with `CL_UISystemCalls`; drives the menu VM each frame.
- **`code/server/sv_game.c`** — calls `VM_Create` with `SV_GameSystemCalls`; invokes `VM_Call(gvm, GAME_RUN_FRAME, ...)` every server tick; hosts the authoritative simulation.
- **`code/client/cl_cgame.c`, `code/cgame/cg_syscalls.c`** — the `trap_*` wrappers inside cgame call back through `VM_DllSyscall` → `currentVM->systemCall` to reach the engine.
- **`code/qcommon/vm_x86.c`, `vm_ppc.c`, `vm_ppc_new.c`** — implement `VM_Compile`/`VM_CallCompiled`; tightly coupled to `vm_local.h` types defined alongside this file.
- **`code/qcommon/vm_interpreted.c`** — implements `VM_PrepareInterpreter`/`VM_CallInterpreted`; the software fallback.
- **`currentVM` global** is read by `VM_ArgPtr` and `VM_ExplicitArgPtr` to bounds-check VM pointers on behalf of any subsystem that processes VM-originated data.

### Outgoing (what this file depends on)

- **Filesystem** (`FS_ReadFile`, `FS_FreeFile`) — loads `.qvm` images and `.map` symbol files.
- **Memory** (`Hunk_Alloc` for permanent VM data/symbols; `Z_Malloc`/`Z_Free` for VM struct itself per first-pass doc) — reflects the engine's hunk-vs-zone lifecycle model.
- **Platform DLL loader** (`Sys_LoadDll`, `Sys_UnloadDll` in `win32/win_main.c` or `unix/unix_main.c`) — the native path bypasses the bytecode entirely.
- **CVar system** (`Cvar_Get`) — reads `vm_cgame`, `vm_game`, `vm_ui` to determine execution mode; reads `com_developer` to gate symbol loading; reads `fs_restrict` in `VM_Create` for demo-mode coercion.
- **Command system** (`Cmd_AddCommand`) — registers `vmprofile` and `vminfo` as developer console commands.
- **`com_developer`** — a global cvar pointer read at runtime; symbol table loading is entirely gated behind it.

## Design Patterns & Rationale

- **Strategy pattern** for the execution backend: `VMI_NATIVE`, `VMI_COMPILED`, `VMI_BYTECODE` are runtime-selectable strategies behind the same `VM_Call` interface. The engine never needs to know which mode a given VM uses after creation.
- **Context pointer as implicit parameter**: `currentVM` is a saved/restored global rather than a passed parameter. This is an intentional ABI compatibility decision — the DLL entry point (`entryPoint`) and the syscall callback (`VM_DllSyscall`) both need access to the current VM but cannot receive it as a parameter through their fixed signatures. This pattern makes the system non-reentrant by design: Q3's single-threaded execution model is assumed throughout.
- **Integer-only ABI**: All VM calls and syscalls use `int *` argument arrays. This is the central constraint that shapes every `trap_*` function in cgame/game/ui. Floats are passed as their bit-reinterpretation as `int` (via union or cast). This choice was forced by the need for a uniform ABI across native DLLs, compiled bytecode, and the software interpreter.
- **Fixed object pool** (`vmTable[3]`): No dynamic allocation for the `vm_t` slots themselves — they are embedded in a static array, zeroed on `VM_Init`, and reclaimed by name-clearing. Simplicity over generality; the game always has exactly three VMs.
- **Demo-mode forced compilation** (`fs_restrict` → `VMI_COMPILED`): Native DLLs could bypass content restrictions; forcing bytecode mode keeps demo builds sandboxed without a separate build path.

## Data Flow Through This File

```
Disk (.qvm file)
  → FS_ReadFile → vmHeader_t
  → LittleLong byte-swap
  → validate magic/lengths
  → dataLength rounded to power-of-2 (for dataMask)
  → Hunk_Alloc(dataLength + STACK_SIZE)  → vm->dataBase
  → Hunk_Alloc(instructionCount * 4)     → vm->instructionPointers
  → VM_Compile (x86 JIT) or VM_PrepareInterpreter
  → VM_LoadSymbols (developer only)
  → vm_t ready in vmTable[]

Per-frame call path:
  Engine subsystem
    → VM_Call(vm, callnum, args...)
    → save/set currentVM
    → vm->entryPoint(args) [DLL]
    OR VM_CallCompiled(vm, args)
    OR VM_CallInterpreted(vm, args)
    → VM executes, issues trap_* calls
    → trap_* → VM_DllSyscall(arg, ...)
    → currentVM->systemCall(&arg)  [e.g. CL_CgameSystemCalls / SV_GameSystemCalls]
    → engine service executes
    → return value propagates back
    → restore currentVM
    → return int to engine subsystem
```

The `dataMask` is the key sandbox mechanism: in interpreted/compiled mode, every VM pointer is ANDed against it before dereferencing, constraining all VM memory access to the allocated data image. In native DLL mode, the mask is bypassed entirely — DLL VMs have full host process memory access, which is the explicit tradeoff accepted for performance.

## Learning Notes

- **Early bytecode sandboxing**: This is a pre-WASM, pre-Lua approach to mod safety. The Q3 QVM is a custom RISC-like ISA specifically designed to be safely interpreted in a host process. The `dataMask` trick (power-of-2 allocation + bitwise AND) is a cheap alternative to full memory-mapped sandboxing.
- **Hunk allocation for permanence**: Symbols and VM code are `Hunk_Alloc`'d (permanent), not `Z_Malloc`'d (frame-temporary). This reflects Q3's dual-allocator model: the hunk is a linear bump allocator that persists until a map change; zone is for short-lived allocations. `VM_Restart` is the mechanism that avoids re-doing hunk allocations on `map_restart`.
- **Platform varargs hazard**: The `VM_DllSyscall` comment is a mini-lecture on x86 vs PPC calling conventions and why `&arg` works on x86 (stack-passed varargs adjacent in memory) but fails on PPC (register-passed varargs not contiguous). Modern engines solve this with explicit struct-based ABIs.
- **No ECS, no scene graph**: The VM boundary is a pure function-call API. There is no shared data structure or component system crossing the boundary — all data exchange goes through integer-typed syscall arguments and the `entityState_t`/`playerState_t` structs marshalled through snapshot machinery in the server/client layers.
- **Profiling hooks in the debug layer**: `VM_VmProfile_f` and the `vmSymbol_t.profileCount` field represent a lightweight sampling profiler built into the engine — a rarity in 1999-era game engines and a sign that id took VM performance measurement seriously.

## Potential Issues

- **`currentVM` is not thread-safe**: If the renderer's SMP path (`tr_backend.c`) ever triggered a VM call (it does not, but the pattern is fragile), `currentVM` would race. The architecture implicitly assumes all VM calls occur on the main thread.
- **Static buffers in symbol lookup**: `VM_ValueToSymbol` uses a `static char text[MAX_TOKEN_CHARS]` return buffer — callers that store the pointer and call again will silently get overwritten data. This is a classic C-era re-entrancy hazard.
- **`VM_VM2C` is a no-op stub**: The function performs only an identity cast with no bounds verification. Any code calling it assumes the caller has already validated the pointer, which is easy to violate in practice.
- **`VM_Restart` logs a stale `filename`**: `Com_Printf("VM_Restart()\n", filename)` passes `filename` before it is initialized — a harmless format-string bug (the `%s` substitution is absent from the format string) but indicative of copy-paste error.
