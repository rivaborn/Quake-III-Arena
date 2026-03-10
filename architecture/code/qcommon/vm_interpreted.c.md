# code/qcommon/vm_interpreted.c

## File Purpose
Implements the software interpreter backend for the Quake III Q3VM virtual machine. It prepares bytecode for interpreted execution and runs the fetch-decode-execute loop over Q3VM instructions, supporting recursive entry and system call dispatch.

## Core Responsibilities
- Translate raw Q3VM bytecode into an int-aligned code image, resolving branch targets to absolute code offsets
- Execute Q3VM instructions via a central dispatch loop using a software operand stack
- Dispatch negative program counters as system calls to the engine
- Enforce VM sandboxing via `dataMask` on all memory accesses
- Support recursive VM entry (reentrant interpreter state per call)
- Provide debug utilities: stack trace, call indentation, opcode name table (when `DEBUG_VM` is defined)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `opcode_t` | enum (defined in `vm_local.h`) | All Q3VM instruction opcodes |
| `vm_t` / `vm_s` | struct (defined in `vm_local.h`) | Per-VM state: code/data image, stack, symbol table, system call pointer |
| `vmSymbol_t` | struct (defined in `vm_local.h`) | Debug symbol entry (name + profile counter) |
| `vmHeader_t` | struct (defined elsewhere) | On-disk/in-memory Q3VM file header with code/data offsets |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `opnames` | `char *[256]` | static (file, `DEBUG_VM` only) | Human-readable opcode name strings for debug logging |

## Key Functions / Methods

### VM_PrepareInterpreter
- **Signature:** `void VM_PrepareInterpreter( vm_t *vm, vmHeader_t *header )`
- **Purpose:** Converts raw bytecode into an int-aligned `codeBase` array; in a second pass, rewrites branch target operands from instruction indices to absolute byte offsets in `codeBase`.
- **Inputs:** `vm` — VM instance to prepare; `header` — points to loaded Q3VM bytecode.
- **Outputs/Return:** None. Mutates `vm->codeBase` and `vm->instructionPointers`.
- **Side effects:** Calls `Hunk_Alloc` to allocate `vm->codeBase` (permanent hunk). Reads `header->codeOffset` region. Calls `Com_Error(ERR_FATAL)` on pc overrun.
- **Calls:** `Hunk_Alloc`, `Com_Error`, `loadWord` (macro/inline).
- **Notes:** Two-pass design: pass 1 copies opcodes and byte-swaps multi-byte operands into int slots; pass 2 resolves branch targets via `vm->instructionPointers[]`. Only branch opcodes (OP_EQ … OP_GEF) have their operands rewritten; OP_JUMP is resolved at runtime.

### VM_CallInterpreted
- **Signature:** `int VM_CallInterpreted( vm_t *vm, int *args )`
- **Purpose:** Executes the VM starting at instruction 0 with up to 10 arguments, running until an OP_LEAVE with PC == -1 is reached (the synthetic return sentinel pushed during setup).
- **Inputs:** `vm` — VM to run; `args[0..9]` — call arguments placed on the VM data stack.
- **Outputs/Return:** `int` — top of operand stack on exit (the function's return value).
- **Side effects:** Sets/clears `vm->currentlyInterpreting`. Writes args and sentinels into `vm->dataBase`. Saves/restores `vm->programStack`. Calls `vm->systemCall` for negative PCs. Calls `Com_Error(ERR_DROP)` on stack overflow, underflow, misalignment, or bad instruction. Calls `VM_Debug(0)`.
- **Calls:** `vm->systemCall`, `vm->instructionPointers[]` (indirect), `Com_Error`, `Com_Printf` (debug), `VM_ValueToFunctionSymbol`, `VM_ValueToSymbol`, `VM_Debug`.
- **Notes:** Uses a local `stack[256]` as the operand stack (not the VM data stack). `dataMask` applied to every load/store address for sandbox enforcement. `r2` is a macro reading the next int from `codeImage[programCounter]` — used as the immediate operand without advancing PC (the handler itself advances). Reentrant: saves `vm->programStack` and `vm->callLevel` around system calls.

### VM_Indent
- **Signature:** `char *VM_Indent( vm_t *vm )`
- **Purpose:** Returns a pointer into a whitespace string to produce call-depth indentation for debug output.
- **Notes:** Caps at depth 20.

### VM_StackTrace
- **Signature:** `void VM_StackTrace( vm_t *vm, int programCounter, int programStack )`
- **Purpose:** Walks the saved return-address chain in `vm->dataBase` and prints up to 32 frames via `Com_Printf`.

## Control Flow Notes
- **Init:** `VM_PrepareInterpreter` is called once after a Q3VM file is loaded (from `vm.c`) when the compiled backend is unavailable or disabled.
- **Per-call:** `VM_CallInterpreted` is the entry point used by `VM_Call` in `vm.c`. It is reentrant — system calls can trigger another `VM_Call`, which re-enters here with a lower `programStack`.
- **Frame:** The interpreter loop runs entirely within a single host call; there is no per-frame coroutine yield.
- **Shutdown:** No explicit shutdown; VM state is freed by the owning `vm.c` module.

## External Dependencies
- **Includes:** `vm_local.h` → `q_shared.h`, `qcommon.h`
- **Defined elsewhere:**
  - `vm_t`, `vmHeader_t`, `vmSymbol_t`, `opcode_t` — `vm_local.h`
  - `Hunk_Alloc`, `Com_Error`, `Com_Printf` — engine common layer
  - `VM_ValueToSymbol`, `VM_ValueToFunctionSymbol`, `VM_LogSyscalls`, `VM_Debug` — `vm.c`
  - `currentVM`, `vm_debugLevel` — `vm.c`
  - `loadWord` — macro; on PPC uses `lwbrx` byte-reverse load; on other platforms a plain int dereference
