# code/unix/vm_x86.c

## File Purpose
This is the Linux/Unix x86-specific stub for the Quake III Virtual Machine (Q3VM) JIT compiler. It provides empty placeholder implementations of `VM_Compile` and `VM_CallCompiled`, indicating the x86 JIT backend was not implemented (or not yet ported) for this Unix target.

## Core Responsibilities
- Satisfies the linker requirement for `VM_Compile` and `VM_CallCompiled` on Unix/x86 builds
- Acts as a no-op stub — the Unix build falls back to the interpreted VM path (`VM_CallInterpreted`) rather than JIT-compiled execution
- Mirrors the interface contract declared in `vm_local.h`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vm_t` | struct (`vm_s`) | VM instance: holds program stack, code/data buffers, symbol table, JIT state flags |
| `vmHeader_t` | struct (defined in `qcommon.h`) | Header parsed from a `.qvm` bytecode file; used as input to compilation |

## Global / File-Static State

None.

## Key Functions / Methods

### VM_Compile
- **Signature:** `void VM_Compile( vm_t *vm, vmHeader_t *header )`
- **Purpose:** Stub. On a real JIT backend (e.g., `code/qcommon/vm_x86.c` or the Win32 counterpart), this would translate Q3VM bytecode into native x86 machine code and store it in `vm->codeBase`.
- **Inputs:** `vm` — target VM instance; `header` — parsed `.qvm` bytecode header
- **Outputs/Return:** `void`
- **Side effects:** None (empty body)
- **Calls:** None
- **Notes:** Because this is a no-op, `vm->compiled` will never be set to `qtrue` through this path; the engine falls through to the interpreter.

### VM_CallCompiled
- **Signature:** `int VM_CallCompiled( vm_t *vm, int *args )`
- **Purpose:** Stub. On a real JIT backend this would dispatch execution into previously JIT-compiled native code, managing the program stack and system call trampoline.
- **Inputs:** `vm` — target VM instance; `args` — array of integer arguments for the VM entry call
- **Outputs/Return:** `int` — undefined (no return statement; UB in practice, but never called because `VM_Compile` is a no-op)
- **Side effects:** None (empty body)
- **Calls:** None
- **Notes:** Should never be reached at runtime on this platform since `VM_Compile` does not set `vm->compiled`.

## Control Flow Notes

This file plays no role in the normal frame loop. During VM initialization (`VM_Create` in `qcommon/vm.c`), the engine calls `VM_Compile` if a native JIT path is desired. Because this stub does nothing, the engine will instead use `VM_PrepareInterpreter` / `VM_CallInterpreted` for all VM dispatch on Unix. This file is relevant only at link time.

## External Dependencies

- **`../qcommon/vm_local.h`** — brings in `vm_t`, `vmHeader_t`, `opcode_t`, `vmSymbol_t`, and the full Q3VM interface declarations
- **`../game/q_shared.h`** (transitively) — base types (`qboolean`, `byte`, `MAX_QPATH`, etc.)
- **`qcommon.h`** (transitively) — `vmHeader_t` definition and common engine declarations
- **Defined elsewhere:** `VM_PrepareInterpreter`, `VM_CallInterpreted`, `currentVM`, `vm_debugLevel` — all implemented in `qcommon/vm_interpreted.c` and `qcommon/vm.c`
