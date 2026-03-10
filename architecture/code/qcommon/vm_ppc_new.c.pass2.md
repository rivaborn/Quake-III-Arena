# code/qcommon/vm_ppc_new.c — Enhanced Analysis

## Architectural Role

This file implements the PowerPC JIT compiler backend within the **qcommon VM host subsystem**. It's one of three VM execution engines (`vm_interpreted.c`, `vm_x86.c`, `vm_ppc_new.c`) that `vm.c` selects at load time. The JIT compiler translates Q3 bytecode to native PPC instructions, enabling high-performance execution on PowerPC platforms (primarily macOS and embedded systems). Every call to `game`, `cgame`, or `ui` VMs potentially routes through this code path at runtime via `VM_CallCompiled`.

## Key Cross-References

### Incoming (who depends on this)
- **`qcommon/vm.c:VM_Create`** – Selects this backend and calls `VM_Compile` during `.qvm` load
- **`qcommon/vm.c:VM_Call`** – Invokes `VM_CallCompiled` for each VM call from client/server
- **`code/server/sv_game.c`** – Drives all game VM calls (physics, AI, combat, entity logic) each frame
- **`code/client/cl_cgame.c`** – Drives cgame snapshot processing and prediction each frame
- **`code/client/cl_ui.c`** – Routes all UI menu events through the UI VM
- **`code/botlib` (via trap syscalls)** – `AsmCall` dispatches bot pathfinding requests back to engine

### Outgoing (what this depends on)
- **`qcommon/qcommon.h`** – `vm_t`, `vmHeader_t`, memory allocators (`Z_Malloc`, `Hunk_Alloc`), error dispatch
- **`qcommon/common.c`** – `Com_Error` (fatal compilation errors), `Com_Printf` (debug output)
- **`vm_local.h`** – Opcode enum, VM state, `currentVM` global
- **Inline GCC/CodeWarrior assembly** – `AsmCall` function (platform-specific stack frame, syscall dispatch)
- **`q_shared.c` / `q_math.c`** – For any math utilities used in operand stack management

## Design Patterns & Rationale

**Multi-Pass Compilation with Forward Reference Resolution:**
Pass -1 identifies all branch targets (`jused`); pass 0 emits code and records instruction addresses; pass 1 re-emits with concrete branch offsets. This avoids requiring a separate symbol table or two-phase linking—the compiler is self-contained and delivers native code directly into the hunk buffer.

**Lazy Operand Type Tracking:**
Rather than forcing all operand-stack slots to a fixed type at compile time, the compiler uses `opStackRegType` to track whether each depth is integer or float as execution flows. When a type mismatch occurs (e.g., `OP_ADDF` on an integer), `makeFloat` retroactively patches the earlier load instruction from `LWZ/LWZX` to `LFS/LFSX`, avoiding redundant store-then-reload sequences. This is an elegant peephole optimization unique to dynamic compilation where the code buffer is writable at compile time.

**Register Allocation by Stack Depth:**
The compiler allocates physical PPC GPRs (16–27) and FPRs (0–11) statically to operand stack depths (0–11). This is simpler than graph coloring but assumes maximum depth ≤ 12 (observed empirically in Quake 3). `spillOpStack`/`loadOpStack` save and restore the full set around syscalls to respect ABI volatility constraints.

**Dual Platform Support (GCC vs. CodeWarrior):**
The `AsmCall` function is defined in two forms (see `#else` branches at EOF). GCC uses `asm()` blocks with inline assembly; CodeWarrior uses a standalone `.asm` file. This dual path reflects Quake 3's support for macOS development on both toolchains circa 2005.

## Data Flow Through This File

1. **Compile time (VM load):**
   - Input: Q3 bytecode buffer + `vmHeader_t` metadata
   - Processing: Three-pass scan; emit PPC words to output buffer; backpatch branches
   - Output: `vm->codeBase` (native code), `vm->instructionPointers` (dispatch table), `vm->codeLength`

2. **Runtime (per VM call):**
   - Entry: `VM_CallCompiled(vm, args)` sets up PPC stack frame, populates r3–r10 with args
   - JIT code executes: manipulates operand stack in registers, calls `AsmCall` for syscalls/inter-VM calls
   - Exit: Result written to `stack[1]` by `OP_LEAVE`; returned to caller

3. **Syscall dispatch (inside `AsmCall`):**
   - Negative instruction index → looks up in `vm->systemCall` (engine trap handler)
   - Positive instruction index → looks up in `instructionPointers` (recursive VM call)
   - Return value pushed onto operand stack via `stwu`

## Learning Notes

**What's idiomatic to this era (2000–2005) but rare in modern engines:**
- **Manual register allocation** – Modern JITs use linear-scan or graph coloring over an SSA intermediate representation
- **No optimization passes** – Only peephole-level instruction patching; no constant folding, dead-code elimination, or loop unrolling
- **Inline assembly** – The `AsmCall` dispatcher is pure PPC assembly; today's engines generate IR that's compiled to machine code by LLVM/Cranelift
- **Platform-specific code generation** – Separate backends per CPU architecture; modern engines abstract via LLVM or Cranelift IR
- **Write-after-emit patching** – Retroactively modifying already-emitted instructions (`makeFloat`/`makeInteger`) works because the code buffer is still writable; JIT-compiled pages are typically made executable only at the end

**Architectural significance:**
This file exemplifies the **execute-now VM paradigm** prevalent in 2000s game engines. Unlike modern VMs (Java, V8) with tiered compilation and optimization, Quake 3's JIT was load-once, no-reoptimization. The tradeoff is simplicity and predictability—no warm-up time, but also no runtime specialization based on observed hotspots.

The **ABI boundary crossing** at `AsmCall` is particularly instructive: PPC ABI mandates r1 as the real stack pointer, r3–r10 as parameter registers, and r12–r13 as volatile temps. The JIT respects these constraints to interop with engine code, a form of **binary contract** between managed and native code.

## Potential Issues

- **Stack overflow if depth > 12:** While empirically safe for Quake 3, a pathological bytecode sequence could overflow `opStackIntRegisters` or `opStackFloatRegisters` arrays.
- **Code buffer underestimation:** If estimated code size (`vmHeader_t->codeOffset`) is too small, `compiledOfs` overflows silently; no guard rail beyond manual checks in `VM_Compile`.
- **No dynamic invalidation:** Once compiled and code pages are made executable, the JIT code cannot be patched or garbage-collected. If VMs are freed and reloaded (e.g., mod switching), old code remains orphaned in hunk memory.
- **Platform-specific debuggability:** Without dwarf/stabs debug info, native PPC code is opaque to debuggers; only the bytecode offset (via `instructionPointers`) ties crashes back to Q3 source.
