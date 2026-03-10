# code/qcommon/vm_interpreted.c — Enhanced Analysis

## Architectural Role

This file implements the software interpreter backend for the Q3VM virtual machine, one of three execution strategies available to the VM host (`vm.c`). It acts as a fallback when JIT compilation is unavailable (x86/PPC backends), disabled, or when porting to new architectures. The interpreter is reentrant and stateless per-call, enabling recursive game-logic calls (e.g., cgame → engine → game VM). It enforces VM sandbox boundaries via `dataMask` on all memory accesses, protecting the engine from malicious or buggy game bytecode. The two-pass bytecode preparation resolves branch targets to absolute offsets, enabling O(1) branch execution during interpretation.

## Key Cross-References

### Incoming (who depends on this file)
- **vm.c** (`VM_CallInterpreted`): Main VM host dispatcher calls `VM_CallInterpreted` when JIT is unavailable for cgame, game, or ui VMs
- **System call handlers** in `sv_game.c`, `cl_cgame.c`, `cl_ui.c`: Re-enter the interpreter via negative PC dispatch (e.g., `trap_Trace` call from game → VM → interpreter → syscall → back to interpreter)
- **VM lifecycle** (`VM_Create`, `VM_Load`): `VM_PrepareInterpreter` is invoked once per VM load if compiled backends are disabled

### Outgoing (what this file depends on)
- **vm.c**: `VM_ValueToFunctionSymbol`, `VM_ValueToSymbol`, `VM_Debug`, `VM_LogSyscalls` (debugging/profiling)
- **qcommon engine layer**: `Hunk_Alloc` (permanent allocation for code image), `Com_Error` (fatal/drop errors), `Com_Printf` (debug output)
- **vm.c globals**: `vm->systemCall` (function pointer to dispatch negative PCs), `currentVM`, `vm_debugLevel` (debug mode flags)
- **Implicit type/constant dependencies**: `opcode_t` enum, `vmSymbol_t`, `vm_t` struct (all from `vm_local.h`)

## Design Patterns & Rationale

**1. Two-Pass Bytecode Preparation**
- **Pass 1** copies raw opcodes and byte-swaps multi-byte operands into int-aligned slots; **Pass 2** rewrites branch-target operands from instruction indices to absolute code offsets using `vm->instructionPointers[]`
- **Why**: Enables O(1) branch execution at runtime (load resolved offset directly), vs. searching instruction pointers during execution. This is a classic "static compile-time bytecode optimization" trading memory for speed.

**2. Dual-Stack Architecture**
- Operand stack (`stack[256]`) is ephemeral per-call; data stack (`vm->dataBase[programStack..]`) holds locals/args/frame state
- **Why**: Separates volatile expression evaluation (operand stack) from persistent frame state (data stack), simplifying recursion and frame management

**3. Macro-Based Immediate Operand Loading**
- `#define r2 codeImage[programCounter]` loads the next int without advancing PC; handlers manually advance
- **Why**: Micro-optimization avoiding unnecessary register load in the hot path (fetch-decode loop runs millions of times per frame)

**4. Sandbox Enforcement via Address Masking**
- Every load/store is masked: `address & dataMask` (e.g., `OP_LOAD4: r0 = *(int *)&image[ r0&dataMask ]`)
- **Why**: Critical security boundary; prevents VM code from reading/writing engine memory, even if bytecode is corrupted or malicious

**5. Reentrant State Management**
- System calls save/restore `vm->programStack` and `vm->callLevel`; recursive VM entry is tracked
- **Why**: Allows engine code to call back into VMs (e.g., `trap_*` functions in game VM → engine → cgame VM), maintaining a coherent call stack without explicit coroutines

**6. Goto-Based Dispatch with Two Entry Points**
- `goto nextInstruction` reloads r0/r1 from operand stack; `goto nextInstruction2` skips reload
- **Why**: Avoids redundant memory loads on hot-path (some instructions don't modify the stack top); hand-optimized before compiler branch prediction was sophisticated

## Data Flow Through This File

**Initialization** (`VM_PrepareInterpreter`):
- Input: Raw Q3VM bytecode segment in `header`, instruction count in `header->instructionCount`
- Process:
  1. Allocate `vm->codeBase` as int-aligned memory
  2. Pass 1: Copy opcodes, byte-swap 4-byte immediates into code image, record `vm->instructionPointers[i]` for each instruction
  3. Pass 2: Rewrite branch operands (OP_EQ, OP_NE, OP_LT*, OP_EQF, etc.) from instruction indices to absolute byte offsets using `instructionPointers[]`
- Output: Prepared `vm->codeBase`, populated `vm->instructionPointers[]`

**Per-Call Execution** (`VM_CallInterpreted`):
- Input: Arguments array `args[0..9]`, VM in state `vm->programStack`, `vm->codeBase`, `vm->dataBase`
- Process:
  1. Setup: Push 10 args + return sentinels to data stack frame
  2. Fetch-decode-execute loop: Read opcode at `programCounter`, dispatch via switch, execute (modifying operand stack, data stack, or PC)
  3. Syscall interception: Negative PC triggers return to caller (vm.c) with syscall number; caller dispatches handler, re-enters interpreter
  4. Exit: OP_LEAVE with PC==-1 terminates loop, returns top of operand stack
- Output: Return value (int)

## Learning Notes

**What developers learn from this file:**

1. **Bytecode VM Sandboxing**: How to use address masking as a simple, O(1) boundary enforcer; the importance of masking all memory ops
2. **Interpreter Patterns**: Fetch-decode-execute loop, operand stack discipline, reentrant call management without explicit coroutines
3. **Era-Specific Optimization**: Use of `goto` for dispatch (2001–2005 era, before modern branch prediction and speculative execution); manual register allocation (r0, r1, r2) before compiler auto-vectorization
4. **Bytecode Design Trade-offs**: Fixed-size immediates (4 bytes) simplify preparation but increase code size; instruction pointer array enables fast branch resolution at the cost of extra indirection

**What modern engines do differently:**
- **JIT Compilation**: LuaJIT, Java HotSpot eliminate interpretation overhead; Q3VM's x86/PPC backends hint at this being known-slow
- **Type Specialization**: V8, PyPy specialize bytecode paths per observed type; Q3VM has only one integer type (int, cast to/from float)
- **WASM**: Modern game scripting (WebAssembly) uses structured control flow (blocks, loops) instead of unstructured jumps; enables better SIMD and parallel execution
- **Coroutines**: Lua 5.1 (era-contemporary) had no coroutines; modern Lua and Rust use async/await for cleaner async control flow

**Connections to game engine concepts:**
- **Hot/Cold Split**: Three separate VMs (game, cgame, ui) allow independent tuning and security isolation
- **Determinism**: Bytecode execution is fully deterministic (no floating-point variability, no thread interleaving), enabling network-replayed demos and server reconciliation
- **Latency Budget**: Interpretation adds ~2–3× overhead; critical inner loops (Pmove, collision tests) are shared (`bg_*.c`) and replicated in both game and cgame to avoid syscall cost

## Potential Issues

1. **Unsafe OP_BLOCK_COPY Range Check** (line 433–438)
   - The wrapping logic `count = ((srci+count) & dataMask) - srci` silently truncates `count` if `srci+count` wraps
   - Bytecode specifying a large copy could copy less data than intended, silently corrupting state
   - No validation at preparation time; only detected if data corruption manifests visibly

2. **No Opcode Validation at Preparation**
   - `VM_PrepareInterpreter` does not validate that opcodes are recognized before execution
   - Debug mode has a runtime default case that errors, but release builds silently continue
   - Corrupted/hand-crafted bytecode could trigger undefined behavior

3. **Fixed Operand Stack Depth** (`MAX_STACK = 256`)
   - Any expression deeper than 256 operands overflows, but error is runtime-caught
   - Recursive VMs use the data stack, so nested calls don't trigger this; only deep single-expression evaluation (unlikely in practice)

4. **Reentrant Error Handling Hazard**
   - System calls can invoke `Com_Error`, which uses `longjmp` to unwind to engine error handler
   - If outer `VM_CallInterpreted` context is not fully saved before a syscall, unwinding loses inner VM state
   - Mitigated by careful `vm->currentlyInterpreting` flag, but subtle control-flow risk

5. **Data Masking Does Not Prevent Logic Errors**
   - Masking enforces bounds but does not prevent out-of-order memory access or use-after-free within bounds
   - Bytecode can write to the same address twice in one frame, or read uninitialized local variables
   - Sandbox is *spatial* (address range), not *temporal* (lifetime); suitable for Q3A's memory model but not for modern GC languages
