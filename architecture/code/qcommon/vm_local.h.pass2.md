# code/qcommon/vm_local.h — Enhanced Analysis

## Architectural Role

This header is the **execution sandbox contract** that unifies three fundamentally different execution strategies—native DLL loading, x86/PPC JIT compilation, and software bytecode interpretation—under a single runtime abstraction. It sits at the boundary between the engine core (qcommon) and three independent game/UI modules (game, cgame, ui VMs), enabling the engine to enforce memory isolation (`dataMask`) and versioned syscall ABIs while remaining execution-strategy-agnostic. The `vm_t` structure is polymorphic across three completely different call paths that all converge on the same `VM_CallCompiled` / `VM_CallInterpreted` entry points.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/qcommon/vm.c`** — Main VM lifecycle manager calls `VM_Compile`, `VM_PrepareInterpreter`, sets `currentVM` before dispatching to `VM_CallCompiled` or `VM_CallInterpreted`
- **`code/qcommon/vm_x86.c`, `vm_ppc.c`** — Implement `VM_Compile` and `VM_CallCompiled` with platform-specific native code generation; directly reference `vm_t` field offsets defined by `VM_OFFSET_*` macros for assembly syscall fast-paths
- **`code/qcommon/vm_interpreted.c`** — Implements `VM_CallInterpreted` software dispatch loop that uses `opcode_t` enum to switch on each instruction; references `vm->instructionPointers` built by `VM_PrepareInterpreter`
- **`code/client/cl_cgame.c`, `cl_ui.c`** — Client engine layers call `VM_Call(cgameVM, ...)` and `VM_Call(uiVM, ...)` to drive UI and client-side game logic
- **`code/server/sv_game.c`** — Server calls `VM_Call(gvm, GAME_RUN_FRAME, ...)` for each server frame; routes all trap syscalls through `currentVM`-indexed dispatch
- **All three VMs** (game, cgame, ui) — Compiled from QVM bytecode or linked as native DLLs; must conform to the opcode/ABI implicitly defined by this header

### Outgoing (what this file depends on)

- **`../game/q_shared.h`** — Base types (`qboolean`, `byte`, `MAX_QPATH`)
- **`qcommon.h`** — Forward declarations of `vm_t`, `vmHeader_t`, `vmInterpret_t`; public `VM_Create`/`VM_Call`/`VM_Free` signatures
- **ASM code** — Implicitly, `VM_OFFSET_PROGRAM_STACK` and `VM_OFFSET_SYSTEM_CALL` offsets are baked into x86/PPC JIT trampolines and syscall fast-paths (must not move)

## Design Patterns & Rationale

**Pluggable Execution Engines**: The vm_t structure is opaque to the calling layer (client, server, botlib). The engine selects execution mode at load time (`VMI_NATIVE`, `VMI_COMPILED`, `VMI_INTERPRETED`), then forwards all calls to the same `VM_CallCompiled`/`VM_CallInterpreted` interface. This achieves **late binding of compilation strategy** without duplicating the VM management code.

**Explicit Type System in Bytecode**: The opcode set distinguishes `OP_LTI` (integer less-than) from `OP_LTF` (float less-than) and `OP_SEX8`/`OP_SEX16` (sign-extend). This **eliminates runtime type dispatch overhead** at the cost of larger bytecode; critical for performance-sensitive game loops.

**Fixed-Offset Memory Layout**: The `VM_OFFSET_*` macros pin `programStack` and `systemCall` at fixed offsets so ASM code can access them without register clobbering. This is a **performance micro-optimization** that avoids spilling registers when entering/exiting trap calls.

**Sandbox via Pointer Masking**: The `dataMask` field in vm_t allows the engine to enforce address-space isolation (e.g., VM pointers are masked to `0x0FFFFFFF`, limiting VM data space to 256 MB). This **prevents malicious or buggy VMs from reading/writing engine memory** while remaining cheaper than full page-table isolation.

## Data Flow Through This File

1. **Load Phase**: `vm.c::VM_Create` loads a `.qvm` binary or `.so` DLL, parses `vmHeader_t`, populates `vm_t`
2. **Prepare Phase**: 
   - If compiled: `VM_Compile(vm, header)` → emits native code to `vm->codeBase`
   - If interpreted: `VM_PrepareInterpreter(vm, header)` → builds `vm->instructionPointers` jump table
3. **Execution Phase**: 
   - Caller (engine) sets `currentVM = vm`
   - Calls `VM_CallCompiled(vm, args)` or `VM_CallInterpreted(vm, args)`
   - VM code executes; on trap (e.g., `trap_Trace`), registers syscall via `vm->systemCall` callback
   - Syscall handler uses `currentVM` to translate VM pointers via `VM_ArgPtr(vmaddr) = currentVM->dataBase + (vmaddr & currentVM->dataMask)`
4. **Debug Phase**: After execution, `VM_ValueToSymbol(currentVM, returnAddress)` walks `vm->symbols` for crash stack traces or profiling

## Learning Notes

**Modern Engines vs. Q3**: Modern game engines (Unreal, Unity) achieve VM-like isolation via **OS processes** (separate address spaces, IPC). Quake III's in-process sandbox model is far more efficient for 2000-era hardware but requires careful pointer masking and assumes cooperative (trusted) VM code. This tradeoff is **idiomatic to early-2000s game architecture**.

**Why Stack-Based?**: The QVM is stack-based (not register-based) because the **bytecode was originally compiled by LCC C compiler** (in `lcc/`), which is easier to retarget to stack VMs. Register-based bytecode (like JVM, CLR) is more efficient but harder to generate. Stack-based keeps per-instruction overhead high but per-VM-compile-step minimal.

**Syscall Boundary**: The `systemCall` function pointer in vm_t is the **only bridge** between VM and engine. All engine services (rendering, collision, sound, networking) flow through this single callback. This rigid boundary makes it impossible for VM code to corrupt engine state directly—all interaction is mediated. Critical for **bot VM stability**.

## Potential Issues

- **Symbol Lookup at Runtime**: Walking `vmSymbol_t` singly-linked list on crash is O(n) per symbol; profiling with thousands of VM function calls could be slow. Not critical (profiling is optional) but worth noting.
- **No Version Checking in opcode_t**: If the engine bytecode format changes, old `.qvm` binaries that assume different opcodes will silently corrupt. The protocol is **not self-verifying** (reliant on file timestamps, rebuild coordination).
- **ASM Fast-Path Fragility**: The `VM_OFFSET_*` macros are mirrored in assembly files (`code/unix/vm_x86a.s`, `code/win32/` equivalent). If `vm_s` layout changes, both C and ASM must be updated in sync—easy to miss. No automated consistency check.
