# code/qcommon/vm_x86.c — Enhanced Analysis

## Architectural Role

This file is the x86 JIT backend of qcommon's VM subsystem, sitting between `vm.c` (the lifecycle manager) and all three game module VMs (game, cgame, ui). It implements one of three execution paths that `vm.c` selects at load time: native DLL (`Sys_LoadDll`), x86 JIT (this file), or software interpreter (`vm_interpreted.c`). When chosen, `VM_Compile` runs once at `VM_Create` time, producing executable hunk-resident x86 code; all subsequent invocations flow through `VM_CallCompiled`, which threads ESI/EDI as the VM's private stack pointer and opstack pointer directly through the native ISA — bypassing any interpreter overhead for the entire duration of a game VM call.

The `AsmCall` trampoline is the only bridge back from JIT-land into the engine: it either tail-dispatches to another compiled VM function (positive target, inter-VM call) or invokes `currentVM->systemCall` (negative target, engine syscall). This trampoline is the sole re-entry point from native code into the C engine per call frame.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/qcommon/vm.c`** is the exclusive caller; it calls `VM_Compile` from `VM_Create` and `VM_CallCompiled` from `VM_Call`. No other file calls these two public functions directly.
- **`currentVM`** (defined in `vm.c`, declared extern in `vm_local.h`) is read and written inside `AsmCall` / `callAsmCall` — this global couples the JIT trampoline tightly to `vm.c`'s active-VM tracking.
- **`code/server/sv_game.c`**, **`code/client/cl_cgame.c`**, **`code/client/cl_ui.c`**: all call `VM_Call`, which routes to `VM_CallCompiled` when the JIT path is active. They are indirect callers at runtime.

### Outgoing (what this file depends on)

- **`vm_local.h`** → `vm_t`, `vmHeader_t`, opcode enum, `currentVM` extern, `dataMask` field — the entire bytecode ABI.
- **`code/qcommon/common.c`**: `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memset`, `Com_Memcpy`, `Com_Printf`, `Com_Error` — all memory and error infrastructure.
- **`code/unix/ftol.nasm`** (non-Win32): provides `qftol0F7F`, pointed to by `ftolPtr` and called from JIT-emitted `OP_CVIF` code. The Win32 equivalent is `_ftol` from the CRT.
- **`sys/mman.h`** (non-Win32 include, but the `mprotect` call is `#if 0`'d in released code — see Potential Issues).
- **`callMask`** is set by `VM_CallCompiled` from `vm->dataMask` so the JIT trampoline can bounds-mask opstack values without a C call.

## Design Patterns & Rationale

- **Two-pass compilation with jump fixup**: Pass 0 walks all opcodes to populate `jused[]` (branch targets) and build the `instructionPointers[]` offset table. Pass 1 emits optimized x86 using those offsets to patch backward/forward jumps. This is the minimal viable multi-pass structure; it avoids a separate IR/linking stage by encoding instruction offsets directly into `vm->instructionPointers`.
- **Peephole optimizer as a state machine**: `LastCommand` (an enum) and `pop0`/`pop1` (prior opcodes) implement a 2-token lookahead window. When a new emission would cancel or simplify the prior emission (e.g., `SUB EDI,4` followed by `ADD EDI,4`), the compiler walks back `compiledOfs` and patches `instructionPointers[instruction-1]` in place. This avoids any post-pass fixup at the cost of coupling emitter helpers to global mutable state — a deliberate tradeoff for simplicity.
- **Pointer indirection for JIT callbacks**: `asmCallPtr` and `ftolPtr` are static `int` (function pointer–sized) variables whose addresses are emitted directly into JIT code as 32-bit immediates. This allows the JIT to call back into C without knowing link-time addresses at compile time — the indirection is resolved when `VM_Compile` runs, not at program link time.
- **dataMask sandbox**: Every memory access emitted for `OP_LOAD`/`OP_STORE` ANDs the address with `vm->dataMask` before adding `vm->dataBase`. This is the entire security boundary preventing QVM bytecode from reading or writing outside its sandbox. It is enforced at JIT-emit time, not at runtime.

## Data Flow Through This File

```
vmHeader_t (bytecode image in hunk)
    │
    ▼  VM_Compile (two passes)
    │   Pass 0: pc walk → jused[], instructionPointers[] draft
    │   Pass 1: opcode switch → Emit* helpers → buf[] (Z_Malloc temp)
    │           peephole cancellation via LastCommand / pop0 / pop1
    │           CONST+LOADn, CONST+STOREn, LOCAL+LOCAL+LOAD+OP+STORE fusion
    │
    ├─► Com_Memcpy(buf → Hunk_Alloc) → vm->codeBase (executable hunk region)
    └─► patch instructionPointers[i] += (int)vm->codeBase (absolute addresses)

Per VM_CallCompiled invocation:
    args[] (up to 10 ints)
    │  written into VM dataBase at programStack
    │
    ▼  __asm sets ESI=programStack, EDI=opstack, calls instructionPointers[0]
    │
    │  [native x86 runs; ESI/EDI maintained as VM stack/opstack pointers]
    │
    ▼  OP_CALL → AsmCall trampoline
    │   positive target → call instructionPointers[target] (VM-to-VM)
    │   negative target → currentVM->systemCall(dataBase + programStack + 4)
    │                     return value written to opStack+1
    │
    ▼  ret from entry point
    └─► opStack[1] returned as int result; programStack / opStack integrity checked
```

## Learning Notes

- **Era-idiomatic JIT**: This is a "template JIT" — each opcode maps to a fixed x86 sequence with no register allocation, no SSA, no live-range analysis. Modern JITs (LLVM, V8 TurboFan, JVM JIT) build an IR, perform dataflow analysis, and allocate registers globally. Q3's approach produces suboptimal but correct and predictably fast code.
- **ESI/EDI as dedicated VM registers**: Reserving ESI for the program stack and EDI for the opstack is a calling-convention override: all JIT-compiled code assumes this layout, and `VM_CallCompiled` establishes it via inline asm. This is a classic trick from bytecode JITs of the late 1990s (cf. JVM "zero interpreter" approaches).
- **Security sandbox via mask-and-add**: The `(addr & dataMask) + dataBase` pattern is conceptually similar to Software Fault Isolation (SFI), but far simpler — it requires the data region to be power-of-two aligned. Modern sandboxing (NaCl, Wasm) uses more sophisticated control-flow integrity.
- **Re-entrancy via saved `instructionPointers`**: `VM_CallCompiled` saves and restores the `instructionPointers` static when a VM call recurses (e.g., UI calling back into the engine which calls cgame). This is a lightweight manual save/restore, not a proper green-thread or coroutine.
- **No SIMD, no floating-point registers**: All FP work is x87 stack based (via `ftolPtr` for `OP_CVIF`); there is no SSE usage. This reflects the 1999 target baseline.

## Potential Issues

- **W^X violation**: `vm->codeBase` (allocated via `Hunk_Alloc`) is both writable during `VM_Compile` and later executed. The non-Win32 `mprotect` call to make it executable-only is `#if 0`'d out. On modern Linux/Windows with DEP/NX enforced, this code would fault unless the hunk allocation uses `mmap(PROT_READ|PROT_WRITE|PROT_EXEC)` explicitly — a security gap that post-Q3 open-source forks (`ioquake3`) fix.
- **32-bit only**: All address arithmetic uses `int`-sized function pointers cast to `int`. On x86-64 LP64, pointer truncation silently breaks `asmCallPtr`, `ftolPtr`, and all address emission. This file is unreachable on 64-bit builds without substantial rework.
- **Static state non-reentrancy in `VM_Compile`**: The statics `buf`, `jused`, `compiledOfs`, `pc`, etc. make `VM_Compile` non-reentrant. Calling it from two threads simultaneously would corrupt both compilations. In practice, all three VMs are compiled sequentially at startup, so this is safe — but it is an implicit constraint not enforced by the code.
- **Unix `callProgramStack`/`callOpStack`/`callSyscallNum` race**: If a hypothetical multi-threaded Unix server called two VMs concurrently, the shared statics used to pass state from GCC inline asm into `callAsmCall` would race. The Win32 path avoids this by keeping those values in local stack variables inside the `__declspec(naked)` function.
