# code/qcommon/vm_ppc.c — Enhanced Analysis

## Architectural Role
This file is the PowerPC-specific JIT compiler for the **qcommon VM hosting subsystem** (alongside `vm_x86.c` and `vm_interpreted.c`). It translates Q3VM bytecode into native PPC machine code at load time. Its generated code serves as the hot path for all three production VMs (game, cgame, ui) on PowerPC-based systems, making it a critical performance multiplier in the engine's VM sandbox layer.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/qcommon/vm.c::VM_Create()`** — Called during VM initialization; decides between this JIT, `vm_x86.c`, and the interpreter based on platform
- **Game VMs** (`code/game`, `code/cgame`, `code/ui`, `code/q3_ui`) — Bytecode compiled here at load time; execution flows through `VM_CallCompiled` → generated PPC code → `AsmCall` trampoline
- **Server syscall dispatcher** (`code/server/sv_game.c`) — Receives control when VM syscalls trigger via `AsmCall`
- **Client syscall dispatchers** (`code/client/cl_cgame.c`, `code/client/cl_ui.c`) — Same syscall boundary

### Outgoing (what this file depends on)
- **qcommon infrastructure** (`qcommon.h`, `vm_local.h`) — `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`
- **VM lifecycle** — Reads `vm->codeLength`, `header->instructionCount`, `header->codeOffset`, writes to `vm->codeBase`, `vm->instructionPointers`, `vm->codeLength`
- **Syscall machinery** — Generated code stores `AsmCall` function pointer in r7 and calls it; `AsmCall` dispatches to `vm->systemCall(vm, callnum)` (set by server/client at VM creation)

## Design Patterns & Rationale

**Three-Pass Compilation**
- Pass `-1` (forward scan): Detects branch targets, populates `jused[]` array to suppress peephole optimizations at jump destinations
- Pass `0` (code generation): Allocates hunk buffer, emits code, records instruction pointer offsets
- Pass `1` (backpatch): Fixes up relative branch offsets (PC-relative immediate fields) now that all offsets are known

This avoids naive solutions like pre-allocating branches or using relocation tables.

**Peephole Optimization via `ltopandsecond()`**
The Q3VM opcode stream has redundant stack operations (e.g., `CONST; LOAD4; ADD` pushes then pops the same value). The optimizer checks if the previous emitted instruction was a `STWU` (store-with-update) and the current instruction is not a branch target—if so, it retracts the store and loads directly from the original opstack slot. This pattern appears idiomatic for bytecode-to-native compilers of that era (trading instruction density for peephole smarts).

**Fixed Register Allocation**
PPC has 32 registers; this file reserves them statically: `R_OPSTACK` (r4), `R_STACK` (r3), `R_MEMBASE` (r5), etc. No liveness analysis or register pressure modeling—constraints are tight, so every register is locked to one purpose. This limits optimization but guarantees predictable codegen and avoids allocator complexity.

**Syscall Trampolining**
The `AsmCall` function is called via count register (`mtctr` / `bctrl`) from within generated code. It saves/restores all VM-visible registers before issuing a C call to `vm->systemCall()`. This pattern allows the engine (server/client) to dispatch both intra-VM calls and syscalls through a single boundary point.

## Data Flow Through This File

1. **Input**: Q3VM bytecode stream (`code[0..codeLength-1]`) + `vmHeader_t` metadata
2. **Parsing**: `pc` cursor walks bytecode; `Constant1/4` extract immediates
3. **Code generation**: `Inst/InstImm/Emit4` write 32-bit PPC instruction words into `buf[]`
4. **Optimization checks**: `rtopped`, `pop0/pop1`, `jused[]` track context for peephole decisions
5. **Output**: Finalized buffer → `Hunk_Alloc` → `vm->codeBase`; `instructionPointers[]` maps Q3VM instruction indices to byte offsets in native code
6. **Execution**: `VM_CallCompiled` sets up stack frame, calls native code function pointer; generated code calls back via `AsmCall` for syscalls

## Learning Notes

**Idiomatic to This Engine / Era**
- **Multi-pass JIT on limited hardware**: Three-pass compilation was practical when memory and compile time were precious. Modern JITs use one-pass with relocation metadata.
- **Bytecode interpreter fallback**: The presence of `vm_interpreted.c` alongside platform-specific JITs shows pragmatic fallback strategy for unsupported architectures.
- **Inline assembly for ABI boundaries**: Both GCC inline asm (`__asm__`) and Metrowerks `asm` blocks in the same file reflect cross-compiler portability (pre-LLVM era).
- **Explicit register allocation**: Unlike RISC V or Arm64 with abundant registers, PPC on GameCube/Xbox still had tight constraints, so every register is named and locked.

**Modern Equivalents**
- TurboFan (V8) or Cranelift would use intermediate representations (SSA) and perform liveness analysis at compile time.
- LLVM JITs would leverage LLVM's register allocator rather than hand-rolling.
- However, the **three-pass strategy** remains sound: e.g., Java's C1 compiler also uses multiple passes.

**Engine-specific Architecture**
- The VM sandbox enforces `dataMask` on all memory accesses (see first-pass `OP_LOAD*` peephole), preventing out-of-bounds reads into engine code/data.
- Syscalls are signed integers: negative values trigger engine dispatch; non-negative values trigger intra-VM jumps. This encoding is enforced by `AsmCall` logic.

## Potential Issues

1. **Immediate Overflow**: `InstImm` calls `Com_Error(ERR_FATAL)` if a signed 16-bit immediate overflows, and `InstImmU` if unsigned overflows. Large opcode constants or stack offsets could trigger this, though the 8× buffer sizing and short opcode lengths make it unlikely in practice.

2. **Branch Target Accuracy**: The `jused[]` array is only used in `ltopandsecond()` to suppress peephole optimization at jump targets. If a branch target is miscalculated in pass 0, pass 1's backpatching may corrupt nearby instructions—but this appears mitigated by the three-pass design.

3. **Undocumented State** (`pop0`, `pop1`, `oc0`, `oc1`): These are initialized to sentinel values but never checked. They appear to be remnants of planned peephole logic that was never implemented. Harmless but confusing.

4. **`itofConvert` Magic Constants**: Integer-to-float conversion uses hardcoded double constants (`0x43300000`, `0x80000000`). These are IEEE 754 magic values (2^52, signed bit mask) that assume specific FPU behavior. Comment documenting the trick would help maintainability.

---

**Sources & Connections**
- **First-pass doc**: Established file responsibilities, opcode coverage, register allocations
- **Architecture context**: Revealed VM hosting as subsystem; game/cgame/ui are the three consumers
- **Cross-refs**: Showed syscall dispatch boundary (server/client) and hunk allocation infrastructure
