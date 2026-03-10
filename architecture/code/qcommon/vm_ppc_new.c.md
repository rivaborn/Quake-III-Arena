# code/qcommon/vm_ppc_new.c

## File Purpose
Implements a PowerPC JIT compiler for Quake III's bytecode virtual machine. It translates Q3 VM bytecode (`vmHeader_t`) into native PPC machine code at load time, then provides an entry point (`VM_CallCompiled`) to execute that code natively.

## Core Responsibilities
- Translate Q3 VM opcodes to PPC machine instructions in a multi-pass compile loop
- Manage a virtual operand stack using physical PPC integer and float registers
- Emit properly encoded PPC instruction words (I-form, D-form, X-form, etc.)
- Patch load instructions retroactively to switch between integer (`LWZ/LWZX`) and float (`LFS/LFSX`) variants as operand types are resolved
- Set up and tear down the native stack frame on VM entry/exit (`OP_ENTER`/`OP_LEAVE`)
- Handle both VM-to-VM calls and VM-to-system-trap calls via `AsmCall`
- Provide `AsmCall` in inline GCC assembly (or CodeWarrior assembly) to dispatch both call types

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `regNums_t` | enum | Logical names for PPC register assignments (stack, opstack, membase, etc.) |
| `ppcOpcodes_t` | enum | Encoded base opcodes for all PPC instructions used during emission |
| `opcode_t` | enum (vm_local.h) | Q3 VM bytecode opcode set |
| `vm_t` | struct (vm_local.h) | VM instance: code/data buffers, instruction pointer table, system call ptr |
| `vmHeader_t` | struct (qcommon.h) | Header of a `.qvm` bytecode file: code offset, instruction count, etc. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `buf` | `unsigned *` | static | Output buffer for emitted PPC words during compilation |
| `compiledOfs` | `int` | static | Current write offset into `buf` (in dwords) |
| `pass` | `int` | static | Compilation pass counter (-1, 0, 1) |
| `code` | `byte *` | static | Pointer into the bytecode stream being compiled |
| `pc` | `int` | static | Current bytecode read position |
| `opStackIntRegisters` | `int[12]` | static | PPC GPR assignments for each operand stack depth slot |
| `opStackFloatRegisters` | `int[12]` | static | PPC FPR assignments for each operand stack depth slot |
| `opStackRegType` | `int[12]` | static | Type tag per stack slot: 0=empty, 1=integer, 2=float |
| `opStackLoadInstructionAddr` | `unsigned *[12]` | static | Address of the load instruction for each stack slot, for retroactive patching |
| `itofConvert` | `double[2]` | static (file) | Magic constants for integer-to-float conversion via bit manipulation |
| `jused` | `byte *` | static | Per-instruction flag marking branch targets (used to guard peephole optimizations) |
| `tvm` | `vm_t *` | static | Current VM being compiled |
| `instruction` | `int` | static | Current instruction index during compilation |

## Key Functions / Methods

### Emit4
- **Signature:** `static void Emit4(char *opname, int i)`
- **Purpose:** Writes a raw 32-bit word into `buf` at `compiledOfs`.
- **Inputs:** Opname (debug only), raw instruction word.
- **Outputs/Return:** None.
- **Side effects:** Increments `compiledOfs`; prints debug info on pass 1.
- **Calls:** None.

### Inst / Inst4 / InstImm / InstImmU
- **Signature:** Various static void helpers.
- **Purpose:** Encode and emit specific PPC instruction forms (register-register, register-immediate, four-register). `InstImm`/`InstImmU` validate 16-bit immediate range.
- **Side effects:** Write to `buf[compiledOfs++]`; `InstImm` calls `Com_Error` on out-of-range immediate.

### spillOpStack / loadOpStack
- **Signature:** `static void spillOpStack(int depth)` / `static void loadOpStack(int depth)`
- **Purpose:** Flush all live operand-stack registers to the in-memory opstack before a `CALL`, then reload them after return. Necessary because `AsmCall` may clobber volatile registers.
- **Side effects:** Emits `stw`/`stfs` (spill) or `lwz` (load) instructions; updates `opStackRegType` and `opStackLoadInstructionAddr`.

### makeFloat / makeInteger
- **Signature:** `static void makeFloat(int depth)` / `static void makeInteger(int depth)`
- **Purpose:** Retroactively patch an earlier load instruction from integer (`LWZ/LWZX`) to float (`LFS/LFSX`) or vice versa, avoiding redundant store/reload sequences. Falls back to an explicit store-then-load when no patchable load address is recorded (e.g., constants).
- **Side effects:** Modifies already-emitted instruction words in `buf`; emits NOP padding (`ori 0,0,0`) when a pipeline hazard requires a dispatch-group boundary.

### VM_Compile
- **Signature:** `void VM_Compile(vm_t *vm, vmHeader_t *header)`
- **Purpose:** Three-pass JIT compiler entry point. Pass -1 identifies branch targets (`jused`). Pass 0 emits code and records instruction pointers. Pass 1 re-emits with resolved branch offsets in place.
- **Inputs:** `vm` – destination VM struct; `header` – bytecode file header.
- **Outputs/Return:** Populates `vm->codeBase`, `vm->codeLength`, `vm->instructionPointers`.
- **Side effects:** `Z_Malloc`/`Hunk_Alloc`/`Z_Free`; calls `Com_Printf`; calls `Com_Error` on overflow or bad opcode; writes native code into the hunk.
- **Calls:** `Constant4`, `Constant1`, `Emit4`, `Inst`, `Inst4`, `InstImm`, `InstImmU`, `spillOpStack`, `loadOpStack`, `makeFloat`, `makeInteger`, `Com_Error`, `Com_Printf`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memcpy`.
- **Notes:** `OP_CONST` followed by `OP_JUMP` marks `jused` for that target. The first `OP_ENTER` encountered is treated as the main VM entry and saves/restores the full operand-stack register set to comply with the PPC ABI.

### VM_CallCompiled
- **Signature:** `int VM_CallCompiled(vm_t *vm, int *args)`
- **Purpose:** Sets up the VM data stack frame with up to 10 arguments, then jumps into the JIT-compiled native code.
- **Inputs:** `vm` – compiled VM; `args` – array of integer arguments.
- **Outputs/Return:** Integer return value from `stack[1]` (written by `OP_LEAVE` via `AsmCall`).
- **Side effects:** Sets `currentVM`; modifies `vm->programStack` and `vm->currentlyInterpreting`; passes 8 parameters in r3–r10 per PPC ABI.
- **Notes:** GCC vs. CodeWarrior differ in how `AsmCall`'s address is passed (direct vs. dereferenced function pointer header).

### AsmCall
- **Signature:** `void AsmCall(void)` — pure assembly (GCC inline asm or CW asm function)
- **Purpose:** Called by JIT-compiled code to dispatch either a VM-internal call (index ≥ 0, looked up in `instructionPointers`) or a system trap (index < 0, dispatched through `vm->systemCall`). Saves and restores all VM registers around system traps.
- **Side effects:** Saves r3–r10 + LR onto the PPC real stack; calls `vm->systemCall`; pushes return value onto the operand stack (`stwu r12,4(r4)`).

## Control Flow Notes
- Called during VM load time from `VM_Compile` (not per-frame).
- `VM_CallCompiled` is the runtime entry invoked each time game code calls into the VM (equivalent to the `vmMain` dispatch).
- `AsmCall` is the only re-entrant path back from JIT code into the engine, used for both inter-VM calls and syscalls.

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h`: `vm_t`, `vmHeader_t`, opcode enum, `Com_Error`, `Com_Printf`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memcpy`, `currentVM`
- `AsmCall` — declared `extern void AsmCall(void)` and defined at file bottom; referenced by `VM_Compile` (address stored in r7/`R_ASMCALL` at runtime)
- `itofConvert` — file-static but referenced by address inside JIT-emitted code for `OP_CVIF`
