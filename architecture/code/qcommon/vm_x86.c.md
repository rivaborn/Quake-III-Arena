# code/qcommon/vm_x86.c

## File Purpose
Implements a load-time x86 JIT compiler for Quake III's virtual machine bytecode. It translates Q3VM opcodes into native x86 machine code at load time, and provides the entry point for calling into the compiled code.

## Core Responsibilities
- Translate Q3VM bytecode (`vmHeader_t`) into native x86 machine code (`VM_Compile`)
- Perform peephole optimizations during two-pass compilation (e.g., folding CONST+LOAD, eliding redundant stack moves)
- Manage the `AsmCall` trampoline for VM-to-syscall and VM-to-VM dispatch
- Execute compiled VM code via `VM_CallCompiled`, setting up the program stack and opstack
- Track jump targets (`jused[]`) to prevent optimizations that cross branch destinations
- Handle cross-platform differences (Win32 `__declspec(naked)` vs GCC inline asm)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ELastCommand` | enum | Tracks the last emitted instruction for peephole optimization / cancellation |
| `vm_t` | struct (defined in vm_local.h) | The VM instance; holds codeBase, dataBase, instructionPointers, etc. |
| `vmHeader_t` | struct (defined elsewhere) | Header of the Q3VM bytecode image; codeOffset, codeLength, instructionCount |
| `opcode_t` | enum (vm_local.h) | All Q3VM opcodes translated in the switch |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `buf` | `byte *` | static | Temporary output buffer for emitted x86 bytes during compilation |
| `jused` | `byte *` | static | Per-instruction flag; marks instructions that are jump targets |
| `compiledOfs` | `int` | static | Current write offset into `buf` |
| `code` | `byte *` | static | Pointer to Q3VM bytecode input |
| `pc` | `int` | static | Program counter into `code` during compilation |
| `instructionPointers` | `int *` | static | Points to active VM's instructionPointers during `VM_CallCompiled` |
| `ftolPtr` | `int` | static | Function pointer to float-to-int conversion (`_ftol` / `qftol0F7F`) |
| `asmCallPtr` | `int` | static | Function pointer to `AsmCall` / `doAsmCall`; called indirectly from JIT code |
| `callMask` | `int` | static | Data mask applied to opstack top after VM-to-VM calls |
| `instruction` | `int` | static | Current instruction index during compilation |
| `pass` | `int` | static | Current compilation pass (0 or 1) |
| `lastConst` | `int` | static | Value of the last `OP_CONST`; used for const-folding |
| `oc0`, `oc1` | `int` | static | Previous two `OP_LOCAL` offsets; used for inc/dec optimization |
| `pop0`, `pop1` | `int` | static | Previous two opcodes; used for peephole decisions |
| `LastCommand` | `ELastCommand` | static | Last emitted command type for peephole cancellation |
| `callProgramStack`, `callOpStack`, `callSyscallNum` | various | static (non-Win32) | Temporaries used by `callAsmCall` to pass state from inline asm |

## Key Functions / Methods

### AsmCall / doAsmCall
- **Signature:** `void AsmCall(void)` (naked / pure asm)
- **Purpose:** Called from JIT-compiled code to dispatch a VM call. If the target is non-negative, jumps to another compiled VM function via `instructionPointers`. If negative, converts to a syscall number and invokes `currentVM->systemCall`.
- **Inputs:** EDI = opstack pointer, ESI = program stack; top-of-opstack holds call target.
- **Outputs/Return:** Places syscall return value on opstack; returns via `ret`.
- **Side effects:** Modifies `currentVM->programStack`; writes syscall number into VM data memory; calls `currentVM->systemCall`.
- **Calls:** `callAsmCall` (non-Win32 path); `currentVM->systemCall` indirectly.
- **Notes:** Win32 version uses `__declspec(naked)` with MSVC inline asm. Non-Win32 uses GCC inline asm; state passed through `callProgramStack`, `callOpStack`, `callSyscallNum` statics.

### VM_Compile
- **Signature:** `void VM_Compile(vm_t *vm, vmHeader_t *header)`
- **Purpose:** Two-pass JIT compiler. Pass 0 maps instruction indices to compiled offsets (and populates `jused`). Pass 1 emits optimized x86 code for each opcode. Copies result to hunk and patches `instructionPointers` to absolute addresses.
- **Inputs:** `vm` — target VM; `header` — bytecode header.
- **Outputs/Return:** void; populates `vm->codeBase`, `vm->codeLength`, `vm->instructionPointers`.
- **Side effects:** Allocates `buf` and `jused` via `Z_Malloc`; frees them after copy. Allocates permanent code on the hunk via `Hunk_Alloc`. Prints compile result via `Com_Printf`.
- **Calls:** `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memset`, `Com_Memcpy`, `Com_Printf`, `Com_Error`, all `Emit*` helpers.
- **Notes:** Peephole optimizations include: CONST+LOAD fusion, CONST+STORE fusion, CONST+ADD/SUB in-place, LOCAL+LOCAL+LOAD4+ADD/SUB+STORE4 → inc/dec, and cancellation of redundant stack adjustments via `LastCommand`.

### VM_CallCompiled
- **Signature:** `int VM_CallCompiled(vm_t *vm, int *args)`
- **Purpose:** Sets up the Q3VM stack frame with up to 10 arguments and jumps into the compiled code entry point.
- **Inputs:** `vm` — the VM to call; `args` — array of integer arguments.
- **Outputs/Return:** Returns the integer value left at the top of the opstack after execution.
- **Side effects:** Sets `currentVM`, `instructionPointers`, `callMask`; modifies `vm->programStack`; saves/restores `oldInstructionPointers` for re-entrant calls.
- **Calls:** Compiled code entry point (indirect); validates opstack and programStack integrity post-call.
- **Notes:** Uses `__asm` (Win32) or `__asm__` (GCC) to set ESI/EDI and call entry point. Validates opStack == `&stack[1]` and programStack == `stackOnEntry - 48` after return.

### Emit helpers
- `Emit1`, `Emit4`: write 1 or 4 bytes into `buf[compiledOfs]`.
- `EmitString`: decodes a hex string (e.g., `"89 07"`) and emits bytes.
- `EmitCommand`: emits a named instruction and records it in `LastCommand`.
- `EmitAddEDI4`, `EmitMovEAXEDI`, `EmitMovEBXEDI`: smart emitters that check `LastCommand` and prior opcodes to cancel or replace redundant sequences.

## Control Flow Notes
- **Init:** `VM_Compile` is called once during VM load (`VM_Create` in `vm.c`), producing native code in the hunk.
- **Per-call:** `VM_CallCompiled` is invoked each time game/cgame/ui code calls into the VM. It is re-entrant (saves/restores `instructionPointers`).
- **Syscalls:** Happen mid-execution via the `AsmCall` trampoline whenever the JIT code executes `OP_CALL` with a negative target.
- No per-frame tick; the JIT code runs to completion each call.

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h`: `vm_t`, `vmHeader_t`, `opcode_t`, `currentVM`, `Com_Error`, `Com_Printf`, `Com_Memcpy`, `Com_Memset`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`
- `sys/mman.h` (non-Win32, `mprotect`) — guarded out in released code (`#if 0`)
- `_ftol` (Win32 CRT) / `qftol0F7F` (Unix NASM, `unix/ftol.nasm`): float-to-int conversion
- `AsmCall` / `doAsmCall`: defined in this file but referenced via `asmCallPtr` indirection so the JIT-emitted `call` uses an indirect pointer fixup
