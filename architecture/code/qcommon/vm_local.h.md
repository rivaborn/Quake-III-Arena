# code/qcommon/vm_local.h

## File Purpose
Internal header defining the data structures, opcodes, and function prototypes for Quake III's Virtual Machine (QVM) system. It is shared by the interpreter (`vm_interpreted.c`), JIT compiler (`vm_x86.c`, `vm_ppc.c`), and core VM manager (`vm.c`).

## Core Responsibilities
- Define the full QVM opcode set used by the bytecode interpreter and compiler
- Declare the `vm_s` (aka `vm_t`) structure holding all runtime state for a VM instance
- Declare the `vmSymbol_t` linked-list structure for debug symbol tracking
- Expose function prototypes for the two execution backends (compiled and interpreted)
- Expose symbol lookup utilities and syscall logging

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `opcode_t` | enum | Complete QVM instruction set (stack ops, branches, arithmetic, memory, type conversions) |
| `vmptr_t` | typedef (int) | VM-internal address/pointer type (offset into VM data/code space) |
| `vmSymbol_t` | struct | Singly-linked list node holding a symbol name, its value, and a profiling counter; variable-length `symName` field |
| `vm_s` / `vm_t` | struct | Full VM instance: execution mode (DLL vs. interpreted vs. compiled), code/data buffers, stack, symbol table, and debug state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `currentVM` | `vm_t *` | global (extern) | Pointer to the VM currently executing; used by syscall trampolines to resolve `VM_ArgPtr` |
| `vm_debugLevel` | `int` | global (extern) | Controls verbosity of VM debug output |

## Key Functions / Methods

### VM_Compile
- Signature: `void VM_Compile( vm_t *vm, vmHeader_t *header )`
- Purpose: JIT-compile a loaded QVM image into native machine code.
- Inputs: Initialized `vm_t`, parsed `vmHeader_t` from the `.qvm` file.
- Outputs/Return: void; populates `vm->codeBase` with native code.
- Side effects: Allocates executable memory; sets `vm->compiled = qtrue`.
- Calls: Defined in `vm_x86.c` or `vm_ppc.c`.
- Notes: Only called when `VMI_COMPILED` interpret mode is requested.

### VM_CallCompiled
- Signature: `int VM_CallCompiled( vm_t *vm, int *args )`
- Purpose: Invoke a compiled VM function, passing arguments on the VM stack.
- Inputs: Active `vm_t` with native code loaded; `args` array.
- Outputs/Return: Integer return value from the VM function.
- Side effects: May re-enter the VM (`callLevel`); invokes `systemCall` callback for trap calls.
- Calls: Defined in `vm_x86.c` / `vm_ppc.c`.

### VM_PrepareInterpreter
- Signature: `void VM_PrepareInterpreter( vm_t *vm, vmHeader_t *header )`
- Purpose: Build the `instructionPointers` jump table for the bytecode interpreter.
- Inputs: `vm_t`, `vmHeader_t`.
- Outputs/Return: void; fills `vm->instructionPointers`.
- Side effects: Allocates `vm->instructionPointers` array.
- Calls: Defined in `vm_interpreted.c`.

### VM_CallInterpreted
- Signature: `int VM_CallInterpreted( vm_t *vm, int *args )`
- Purpose: Execute a QVM function using the software bytecode interpreter.
- Inputs: Active `vm_t`; `args` array.
- Outputs/Return: Integer return value.
- Side effects: Modifies `vm->programStack`; dispatches trap calls via `systemCall`.
- Calls: Defined in `vm_interpreted.c`.

### VM_ValueToFunctionSymbol / VM_SymbolToValue / VM_ValueToSymbol
- Notes: Debug/profiling helpers that walk `vm->symbols` to map between numeric addresses and human-readable symbol names.

### VM_LogSyscalls
- Signature: `void VM_LogSyscalls( int *args )`
- Purpose: Log outgoing system-call arguments for debugging.
- Side effects: I/O (writes to log).

## Control Flow Notes
This header is included only by VM-internal `.c` files. The lifecycle is: `VM_Create` (in `vm.c`) loads a `.qvm` or `.dll`, then calls either `VM_Compile`+`VM_CallCompiled` or `VM_PrepareInterpreter`+`VM_CallInterpreted` depending on `vmInterpret_t`. `currentVM` is set around each call so that `VM_ArgPtr` can translate VM-space pointers for the host engine's syscall handler.

## External Dependencies
- `../game/q_shared.h` — base types (`qboolean`, `byte`, `MAX_QPATH`, etc.)
- `qcommon.h` — `vm_t` forward declaration, `vmInterpret_t`, `vmHeader_t`, `VM_Create`/`VM_Free`/`VM_Call` public API
- `vmHeader_t` — defined in `qfiles.h` (via `qcommon.h`); not defined in this file
- `QDECL` — calling-convention macro from `q_shared.h`
