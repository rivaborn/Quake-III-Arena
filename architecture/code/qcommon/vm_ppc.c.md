# code/qcommon/vm_ppc.c

## File Purpose
Implements a dynamic JIT compiler that translates Quake III VM bytecode (Q3VM opcodes) into native PowerPC machine code at load time. It also provides the runtime entry point (`VM_CallCompiled`) and the system-call trampoline (`AsmCall`) needed to execute the generated code and dispatch engine syscalls.

## Core Responsibilities
- Translate Q3VM opcode stream into raw PPC 32-bit instructions written into a memory buffer
- Perform a multi-pass compile (3 passes: `-1`, `0`, `1`) so forward branch targets resolve correctly
- Emit peephole optimizations (e.g., collapsing CONST+LOAD pairs, eliding redundant stack pushes before binary ops)
- Allocate and finalize the native code buffer on the hunk after pass 0
- Provide `VM_CallCompiled` to set up the VM stack frame and jump into generated code
- Provide `AsmCall` (GCC inline asm or Metrowerks asm) to handle both intra-VM calls and engine syscall dispatch

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `regNums_t` | enum | Symbolic names for PPC register assignments used by the compiler |
| `ppcOpcodes_t` | enum | Encoded PPC instruction primary opcodes (upper bits only; many stubs left at `0x7c000000`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `buf` | `unsigned *` | static | Output buffer for generated PPC instructions |
| `compiledOfs` | `int` | static | Current write position in `buf` (in dwords) |
| `code` | `byte *` | static | Pointer into the Q3VM bytecode being compiled |
| `pc` | `int` | static | Current read offset into `code` |
| `rtopped` | `qboolean` | static | Peephole flag: `qtrue` if `R_TOP` already holds the opstack top |
| `pop0`, `pop1`, `oc0`, `oc1` | `int` | static | Previous opcode and operand history for peephole decisions |
| `tvm` | `vm_t *` | static | VM being compiled (for writing `instructionPointers`) |
| `instruction` | `int` | static | Current bytecode instruction index |
| `jused` | `byte *` | static | Per-instruction flag: `1` if the instruction is a branch target |
| `pass` | `int` | static | Compile pass counter (`-1`, `0`, `1`) |
| `itofConvert` | `double[2]` | global | Magic double constants used for integer-to-float conversion trick |

## Key Functions / Methods

### Constant4 / Constant1
- **Signature:** `static int Constant4(void)` / `static int Constant1(void)`
- **Purpose:** Read a 4-byte or 1-byte little-endian immediate from the bytecode stream.
- **Inputs:** None (reads `code[pc]`, advances `pc`)
- **Outputs/Return:** Integer immediate value
- **Side effects:** Advances `pc`

### Emit4 / Inst / Inst4 / InstImm / InstImmU
- **Purpose:** Write one encoded PPC instruction word into `buf[compiledOfs++]`.
- **Notes:** `InstImm`/`InstImmU` call `Com_Error(ERR_FATAL)` if the immediate overflows 16 bits. These are the only allocation points for the output buffer.

### ltop / ltopandsecond / fltopandsecond
- **Purpose:** Peephole helpers that load the top (and optionally second) opstack value(s) into `R_TOP`/`R_SECOND`, adjusting `R_OPSTACK`. `ltopandsecond` will retract the last `STWU`/`STW` if the previous instruction was a push and the current instruction is not a branch target (`jused[instruction]==0`), saving a redundant store/load pair.
- **Side effects:** May decrement `compiledOfs`; updates `rtopped`.

### VM_Compile
- **Signature:** `void VM_Compile(vm_t *vm, vmHeader_t *header)`
- **Purpose:** Main JIT entry point. Iterates over all Q3VM instructions across 3 passes and emits native PPC code into `buf`. On pass 0, copies the finalized buffer to the hunk and patches `instructionPointers` to absolute addresses. Pass 1 back-patches relative branch offsets.
- **Inputs:** `vm` (destination VM struct), `header` (Q3VM bytecode header with code/data offsets)
- **Outputs/Return:** `void`; populates `vm->codeBase`, `vm->codeLength`, `vm->instructionPointers`
- **Side effects:** `Z_Malloc`/`Z_Free` for temp buffers; `Hunk_Alloc` for final code; `Com_Printf` per pass; `Com_Error` on overflow or unknown opcode
- **Calls:** `Constant4`, `Constant1`, `Inst`, `Inst4`, `InstImm`, `InstImmU`, `Emit4`, `ltop`, `ltopandsecond`, `fltopandsecond`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memset`, `Com_Memcpy`, `Com_Error`, `Com_Printf`

### VM_CallCompiled
- **Signature:** `int VM_CallCompiled(vm_t *vm, int *args)`
- **Purpose:** Sets up the VM stack frame in `vm->dataBase`, then calls the generated native code directly as a C function pointer with 8 register arguments (PPC ABI: r3–r10).
- **Inputs:** `vm`, `args` (up to 10 integer arguments)
- **Outputs/Return:** `stack[1]` — the integer return value left by the VM function
- **Side effects:** Modifies `vm->programStack`, `vm->currentlyInterpreting`; writes into `vm->dataBase`
- **Notes:** Supports recursive VM entry by saving/restoring `programStack`.

### AsmCall
- **Signature:** `void AsmCall(void)` (GCC inline asm or Metrowerks `asm` function)
- **Purpose:** Trampoline called from within generated PPC code at every `OP_CALL`. Pops the call target from the opstack; if non-negative, looks it up in `instructionPointers` and jumps to it (intra-VM call); if negative, converts to a syscall index and dispatches to `vm->systemCall`, saving/restoring all VM registers across the C call.
- **Side effects:** Saves and restores r3–r10, r13, link register; writes `vm->programStack`; performs indirect call to `vm->systemCall`

## Control Flow Notes
`VM_Compile` is called once during VM load (from `vm.c`). After compilation, every call to the VM goes through `VM_CallCompiled`, which directly invokes the native buffer. Within the generated code, `AsmCall` (stored in `R_ASMCALL`/r7 and jumped to via count register) handles the call/syscall boundary. The `OP_LEAVE` opcode generates `bclr` (branch to link register), returning to `VM_CallCompiled`'s call site.

## External Dependencies
- `vm_local.h` → `vm_t`, `vmHeader_t`, `opcode_t` enum, `currentVM`, `vm_debugLevel`
- `q_shared.h` / `qcommon.h` → `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `qboolean`, `byte`
- `AsmCall` — defined in this file, but its address is passed into generated code as a register constant
- `vm->systemCall` — defined elsewhere (engine syscall dispatcher, set at VM creation time)
