# code/qcommon/vm.c

## File Purpose
Implements Quake III Arena's virtual machine management layer, supporting three execution modes: native DLL, compiled QVM bytecode, and interpreted QVM bytecode. It handles VM lifecycle (create, restart, free), cross-boundary call dispatch, symbol table loading, and developer profiling/info commands.

## Core Responsibilities
- Initialize and manage up to 3 simultaneous VM instances (cgame, game, ui)
- Load `.qvm` files from disk, validate headers, and allocate hunk memory for code/data
- Dispatch calls into VMs via `VM_Call`, routing to DLL entry point, compiled, or interpreted backend
- Bridge VM-to-engine system calls via `VM_DllSyscall`
- Translate VM integer pointers to host C pointers with optional mask-based bounds enforcement
- Load and walk `.map` symbol files for developer debugging and profiling
- Expose `vmprofile` and `vminfo` console commands

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `vm_t` (`vm_s`) | struct | Per-VM state: stack, system call pointer, DLL handle, code/data buffers, symbol list, compile flags |
| `vmSymbol_t` | struct | Linked-list node holding a symbol name, its bytecode value, and profiling hit count |
| `vmHeader_t` | struct (defined in qcommon.h) | On-disk `.qvm` file header with magic, segment lengths, instruction count |
| `opcode_t` | enum | Full Q3 VM instruction set (defined in `vm_local.h`, used by backends) |
| `vmInterpret_t` | enum | Execution mode selector: `VMI_NATIVE`, `VMI_COMPILED`, `VMI_BYTECODE` |
| `vmptr_t` | typedef (`int`) | VM-internal pointer type (integer offset into `dataBase`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `currentVM` | `vm_t *` | global | Active VM during a `VM_Call` / syscall; used by `VM_DllSyscall` and `VM_ArgPtr` |
| `lastVM` | `vm_t *` | global | Most recently called VM; used by `vmprofile` command |
| `vm_debugLevel` | `int` | global | Controls debug trace verbosity |
| `vmTable[MAX_VM]` | `vm_t[3]` | file-static (global array) | Pool of all active VM instances |

## Key Functions / Methods

### VM_Init
- **Signature:** `void VM_Init( void )`
- **Purpose:** Registers `vm_cgame`, `vm_game`, `vm_ui` CVars and `vmprofile`/`vminfo` commands; zeroes `vmTable`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies CVar system and command system; zeroes global `vmTable`.
- **Calls:** `Cvar_Get`, `Cmd_AddCommand`, `Com_Memset`
- **Notes:** Called once at engine startup.

### VM_Create
- **Signature:** `vm_t *VM_Create( const char *module, int (*systemCalls)(int *), vmInterpret_t interpret )`
- **Purpose:** Allocates and initializes a VM slot; loads a native DLL or `.qvm` file; compiles or prepares the interpreter; loads symbols.
- **Inputs:** Module name string, engine syscall handler function pointer, desired interpretation mode.
- **Outputs/Return:** Pointer to initialized `vm_t`, or `NULL` on failure.
- **Side effects:** Hunk allocations for `dataBase`, `instructionPointers`; may load a system DLL via `Sys_LoadDll`; calls `VM_Compile` or `VM_PrepareInterpreter`.
- **Calls:** `Hunk_Alloc`, `FS_ReadFile`, `FS_FreeFile`, `Sys_LoadDll`, `VM_Compile`, `VM_PrepareInterpreter`, `VM_LoadSymbols`, `VM_Free`, `Com_Error`
- **Notes:** Deduplicates by name (returns existing VM if already loaded). Demo mode (`fs_restrict`) forces `VMI_COMPILED`. Stack is placed at end of data image; `STACK_SIZE` = 128 KB.

### VM_Restart
- **Signature:** `vm_t *VM_Restart( vm_t *vm )`
- **Purpose:** Reloads `.qvm` data in-place without reallocating memory, enabling `map_restart` without hunk churn. For DLL VMs, fully frees and re-creates.
- **Inputs:** Pointer to an existing `vm_t`.
- **Outputs/Return:** Pointer to the (possibly newly created) `vm_t`.
- **Side effects:** Overwrites `vm->dataBase` content; may call `VM_Free`/`VM_Create` for DLL path.
- **Calls:** `FS_ReadFile`, `FS_FreeFile`, `VM_Free`, `VM_Create`, `Com_Memset`, `Com_Memcpy`, `Com_Error`
- **Notes:** Does not reload code segment or reallocate instruction pointer table.

### VM_Call
- **Signature:** `int QDECL VM_Call( vm_t *vm, int callnum, ... )`
- **Purpose:** Primary engine-to-VM call gate; saves/restores `currentVM`; dispatches to DLL entry point, compiled, or interpreted backend.
- **Inputs:** Target VM, call number, up to 16 variadic int arguments.
- **Outputs/Return:** Integer return value from the VM function.
- **Side effects:** Sets/restores `currentVM` and `lastVM`; may print debug trace.
- **Calls:** `vm->entryPoint` (DLL), `VM_CallCompiled`, `VM_CallInterpreted`
- **Notes:** Argument packing into `args[16]` array works around C variadic calling convention differences across platforms (see `VM_DllSyscall` comment).

### VM_DllSyscall
- **Signature:** `int QDECL VM_DllSyscall( int arg, ... )`
- **Purpose:** Callback registered with `Sys_LoadDll`; native DLL modules call this to invoke engine syscalls.
- **Inputs:** Syscall number as first arg; up to 15 additional int args variadically.
- **Outputs/Return:** Return value of `currentVM->systemCall`.
- **Side effects:** Depends on the engine syscall invoked.
- **Notes:** PowerPC Linux requires explicit `va_list` collection into an array due to register-passing ABI; other platforms pass `&arg` directly.

### VM_Free / VM_Clear
- **Signature:** `void VM_Free( vm_t *vm )` / `void VM_Clear(void)`
- **Purpose:** `VM_Free` unloads a single VM (DLL or QVM) and zeroes its slot. `VM_Clear` iterates all slots and frees any DLL handles.
- **Side effects:** Calls `Sys_UnloadDll`; zeroes `currentVM`/`lastVM`.

### VM_ArgPtr / VM_ExplicitArgPtr
- **Purpose:** Convert a VM integer pointer to a host pointer; apply `dataMask` for interpreted VMs (bounds enforcement), bypass mask for DLL VMs.
- **Notes:** Guard against `NULL` and against missing `currentVM` on reconnect.

### VM_LoadSymbols
- **Purpose:** Parses a `vm/<name>.map` file and builds the `vmSymbol_t` linked list on the hunk; only runs when `com_developer` is set.

- **Notes on trivial helpers:** `VM_ValueToSymbol`, `VM_ValueToFunctionSymbol`, `VM_SymbolToValue`, `VM_SymbolForCompiledPointer` — walk the symbol list for debug/profiling lookups. `ParseHex` — manual hex string parser. `VM_VM2C` — identity cast stub. `VM_Debug` — sets `vm_debugLevel`.

## Control Flow Notes
- **Init:** `VM_Init` is called during `Com_Init`.
- **Per-subsystem load:** `VM_Create` is called by the client (cgame), server (game), and UI modules when those subsystems initialize.
- **Per-frame calls:** `VM_Call` is invoked each frame by the engine to call into cgame/game/ui entry points (e.g., `CG_DrawActiveFrame`, `G_RunFrame`).
- **Shutdown:** `VM_Free`/`VM_Clear` are called on map change or engine shutdown.

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h` — shared types, CVar/Cmd/FS/Hunk/Com APIs
- `Sys_LoadDll` / `Sys_UnloadDll` — platform DLL loader (defined in `sys_*` / `win_main.c` / `unix_main.c`)
- `VM_Compile` / `VM_CallCompiled` — defined in `vm_x86.c` (or `vm_ppc.c`, `vm_ppc_new.c`)
- `VM_PrepareInterpreter` / `VM_CallInterpreted` — defined in `vm_interpreted.c`
- `FS_ReadFile`, `FS_FreeFile`, `Hunk_Alloc`, `Z_Malloc`, `Z_Free`, `Com_Error`, `Com_Printf`, `Cvar_Get`, `Cmd_AddCommand` — all defined in other `qcommon/` modules
