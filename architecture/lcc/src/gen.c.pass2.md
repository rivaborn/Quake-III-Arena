# lcc/src/gen.c — Enhanced Analysis

## Architectural Role

`gen.c` implements the **code generation and register allocation backend** for the LCC compiler's instruction selection phase. It consumes the LBURG-driven tree reductions produced by the pattern matcher, manages the CPU register file throughout code generation, and emits lowered target-machine instructions. This file is critical in the Q3A build pipeline: it processes QVM bytecode compilation for the three runtime VMs (cgame, game, ui), ensuring that generated machine code respects ABI conventions and runtime constraints.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler driver** (`lcc/src/main.c`, `lcc/src/pass2.c`) invokes `gen()` as the final IR-to-machine-code pass after tree parsing and LBURG labeling
- **Platform-specific backends** (`.md` files like `lcc/src/x86.md`) define the `IR` table (`IR->x._rule`, `IR->x._templates`, `IR->x._kids`, etc.) that `gen.c` reads to dispatch instruction emission
- **QVM assembly consumer** (`code/qcommon/vm.c`, `code/qcommon/vm_x86.c`) receives the emitted code as its executable input

### Outgoing (what this file depends on)
- **LBURG machinery** (`lcc/src/tree.c`, `lcc/lburg/lburg.c`): provides `IR->x._rule()`, `IR->x._label()`, cost tables, and reachability predicates (`range()`, NeedsReg[] table)
- **Allocators and memory** (`lcc/src/alloc.c`): `NEW0()`, arena-based symbol/node allocation
- **Platform IR definitions** (`lcc/src/x86.md`, etc.): defines register set masks, register names, instruction templates, and the machine-specific function pointers in the `IR` vtable
- **Math/utility functions** (`lcc/src/dag.c`, `lcc/src/enode.c`): node construction, DAG simplification, symbol creation

## Design Patterns & Rationale

**1. LBURG-Driven Code Selection with BURG Rules**
- Relies on a declarative, bottom-up tree parser (LBURG) to match expression patterns against machine instructions
- Each rule has a cost; `reduce()` selects minimal-cost rules recursively on subtrees
- Why: **Correctness by construction**—BURG guarantees optimal or near-optimal code selection; separation of machine knowledge into `.md` files makes retargeting simpler

**2. Two-Phase Tree Linearization**
- Phase 1: `prelabel()` + `rewrite()` + `reduce()` → decorate DAG with instructions and register constraints
- Phase 2: `ralloc()` → global register allocation over the linearized instruction sequence
- Why: **Separation of concerns**; instruction selection and register allocation are largely independent problems

**3. Register Allocation via Online Spilling**
- `getreg()` / `spillr()` / `genspill()` implement a local, greedy spilling strategy during the linear scan
- Tracks `freemask[IREG]` / `freemask[FREG]` to avoid double-allocation
- Why: **Simplicity and bounded memory**—O(1) spill overhead per conflict, no full graph coloring

**4. Copy Propagation and Move Elimination** (`requate()`)
- Detects chains of move instructions and rewrites them to use source register directly if safe
- Requires liveness analysis via `uses()` and aliasing checks via `setsrc()`
- Why: **Code quality**—avoids redundant register-to-register moves without a full SSA pass

**5. Expression Reuse via CSE Nodes** (`reuse()`)
- If a temporary holds a previous computation (`cse` node) with zero cost to re-evaluate, reuse it instead of reloading
- Gated by `mayrecalc()` on constant-like operations (CNST, ADDRF, ADDRG, ADDRL)
- Why: **Code size reduction**—trades register pressure for fewer loads

## Data Flow Through This File

```
gen(forest)
  ├─ for each node p in forest:
  │   ├─ docall(p) → set syscall arg offset
  │   ├─ rewrite(p)
  │   │  ├─ prelabel(p) → setreg + target constraints
  │   │  ├─ IR->x._label(p) → LBURG state machine setup
  │   │  └─ reduce(p, 1) → recursive pattern matching + cost selection
  │   └─ mark p->x.listed = 1
  │
  ├─ for each node p in forest:
  │   └─ prune(p, &dummy) → remove dead instructions
  │
  └─ ralloc(forest)
      ├─ linearize(forest) → topological sort into x.next chain
      ├─ for each instruction in chain:
      │   ├─ getreg() / spillr() / genspill()
      │   └─ update freemask[], usedmask[]
      └─ emit(forest) → final code generation
          └─ for each instruction:
              ├─ requate() → move elimination
              └─ (*emitter)(p, nt) → emitasm(p) → printf via IR->x._templates
```

**Key state transitions:**
- Stack offsets grow in `mkauto()` via `offset` variable; reset per scope in `blockend()`
- Register masks (`freemask`, `usedmask`) are local to a block; saved/restored via `blockbeg()/blockend()`
- Instruction selection cost is one-way: once `reduce()` assigns a rule to a node, it doesn't backtrack

## Learning Notes

1. **BURG as Architecture**: The LBURG (little Burm UBuRg) tree grammar is the "specification" of code generation; `gen.c` is mostly a **faithful executor** of that spec. Understanding `.md` files is as important as understanding this file.

2. **Register Allocation Trade-off**: This is **not** a full graph-coloring allocator (no coalescing, no interference graphs). Instead, it uses a **linear-scan greedy approach with panic spilling**. Suitable for early-stage compilers and embedded code, but produces code with more spills than modern JITs.

3. **DAG vs. Forest**: The input to `gen()` is a **forest** (list of DAGs), not a single tree. Each top-level statement/expression is a separate DAG; this simplifies both semantics and implementation.

4. **Idiomatic to 1990s Compiler Design**: 
   - No SSA form
   - No explicit liveness analysis (uses ad-hoc `usecount` tracking)
   - No peephole optimization (all code shaping happens in BURG rules and prelabeling)
   - Register coalescing is implicit in `requate()` heuristics, not formalized

5. **Contrast with Modern Engines**:
   - Modern engines (Lua JIT, V8, SpiderMonkey) use **linear scan with interval coloring** or **full graph coloring with advanced heuristics**
   - They also separate **tier-1 baseline** compilation (speed) from **tier-2 optimizing** compilation (throughput)
   - Q3A's VMs don't support dynamic recompilation; this static compilation is a one-shot pass

## Potential Issues

1. **Register Allocation Correctness**
   - The `freemask` tracking assumes no **live-range aliasing**: once `getreg()` claims a register, all subsequent uses of that symbol must use that same register. If a symbol is assigned multiple times, this could break.
   - Mitigation: LBURG rules and `rtarget()` constraints generally enforce single-assignment-like discipline per basic block.

2. **Spilling to Unallocated Stack**
   - `genspill()` calls `(*IR->x.genspill)(...)` which is platform-defined. If the platform allocator doesn't reserve enough frame space (`framesize`), spilled values could corrupt the return address or caller-saved space.
   - Mitigation: `maxoffset` tracking in `blockbeg/blockend` and final frame size computation in `ralloc()` should catch this, but it's not visible in this excerpt.

3. **No Escape Analysis**
   - Functions with unstructured control flow (labels, `goto`) or coroutines could have temporaries live across basic blocks, but the per-block `offset` and `freemask` reset assumes block boundaries.
   - Mitigation: Q3A VMs don't use `goto`; all control is structured (if/loops), so this is likely not an issue in practice.

4. **Hard-Coded NeedsReg[] Table**
   - If a new operation is added to the IR (`opindex`) but NeedsReg[] is not updated, `prelabel()` will silently assign no register, causing silent code corruption.
   - Mitigation: A compile-time assertion or a check in `prelabel()` could catch this, but it's not implemented.
