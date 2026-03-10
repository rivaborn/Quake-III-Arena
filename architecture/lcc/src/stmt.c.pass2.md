# lcc/src/stmt.c — Enhanced Analysis

## Architectural Role

This file is the **statement parser and intermediate-code generator** in the LCC compiler toolchain, which compiles C source code into QVM bytecode for Quake III VMs (cgame, game, ui). While not part of the runtime engine, `stmt.c` is critical infrastructure: every statement in game logic code (`code/game/*.c`, `code/cgame/*.c`) passes through this module during compilation, converting high-level control flow (if/while/switch/for) into a linearized sequence of code objects with labels and branches that the backend will ultimately emit as VM instructions. This is one of the last semantic analysis passes before code generation; it validates statement structure, synthesizes control flow labels, and bridges expression parsing to the VM code emission layer.

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/src/stmt.c` is called by:** `lcc/src/stmt.c` → `statement()` is the entry point invoked recursively by itself and from `lcc/src/decl.c` (function body parsing)
- **Global state read:** `t` (current token from lexer), `token` (token string), `src` (source coordinate), `level` (scope depth), `glevel` (debug level), `cfunc` (current function symbol), `codelist` (code object chain), `stmtlabs` (statement label table)
- **No runtime engine calls:** This is strictly a compile-time tool; the OUTPUT (code objects) are serialized into QVM bytecode files that the runtime engine (`code/qcommon/vm.c`, `code/qcommon/vm_interpreted.c`) will eventually load and execute

### Outgoing (what this file depends on)
- **Internal LCC calls:**
  - `lcc/src/lex.c` → `gettok()`, `getchr()` for tokenization
  - `lcc/src/expr.c` → `expr()`, `expr0()`, `texpr()`, `constexpr()` for expression parsing
  - `lcc/src/gen.c` → `walk()`, `code()` for code object creation
  - `lcc/src/tree.c` → `tree()`, `idtree()`, `cnsttree()`, `eqtree()`, `cast()`, `asgn()`, `rvalue()` for AST manipulation
  - `lcc/src/dag.c` → `listnodes()` for code-list insertion
  - `lcc/src/main.c` → `error()`, `warning()` for diagnostics
  - `lcc/src/symbol.c` → `lookup()`, `install()`, `use()` for symbol table management
  - Helper headers: `c.h` (common types, symbol table), preprocessor macros from compiler infrastructure
- **No direct connection to Quake III engine:** The code objects generated here are serialized via `swgen()`/`swcode()` (switch dispatch optimization) but never call engine subsystems at compile time

## Design Patterns & Rationale

1. **Recursive Descent Parsing:** `statement()` is mutually recursive with helper functions (`ifstmt()`, `forstmt()`, `whilestmt()`, `swstmt()`, `compound()`), implementing a predictive top-down parser where each statement type (IF, FOR, SWITCH, etc.) has a dedicated handler.

2. **Code Object Chain:** Statements emit a linearized chain of `Code` objects (linked list via `codelist`). Each code object represents either:
   - A **Label** (`definelab()`) — jump target
   - **Control flow** (`Jump`, `Branch`) — explicit jumps for break/continue/goto
   - **Switch dispatch** (`Switch`, `Defpoint`) — switch case tables or debug points
   - **Local variable initialization** (`Local`)
   - **Expression evaluation** (`Walk` — executed via `listnodes()`)

3. **Label Synthesis:** Rather than inline jumps, the parser synthesizes temporary labels for each control construct (e.g., `lab` for loop body, `lab+1` for loop continuation, `lab+2` for break target). This decouples the parsing phase from backend label allocation.

4. **Switch Optimization via Density Bucketing:** `swgen()` / `swcode()` implements an **adaptive table-vs.-comparison strategy**: case values are grouped into "buckets" based on density (sparse ranges → comparisons; dense ranges → lookup tables). The `den(i,j)` macro computes density; the recursive `swcode()` subdivides based on midpoint binary search, emitting either equality comparisons or a static jump table.

5. **Reachability Analysis:** `reachable()` checks if code following a Jump or Switch is logically unreachable, issuing warnings to catch dead code. This runs at parse time, not post-hoc analysis.

6. **Refinement Counter (`refinc`):** Branches adjust the `refinc` variable to weight code coverage estimates (e.g., `refinc /= 2.0` in if-branches, `refinc *= 10.0` in loop init). This is used for profiling/instrumentation feedback during compilation.

## Data Flow Through This File

1. **Input:** Token stream from lexer (`t`, `token`, `src`); symbol tables (`cfunc` = current function).
2. **Processing:**
   - `statement()` dispatches on token type (IF, WHILE, FOR, etc.)
   - Each handler parses sub-expressions and nested statements recursively
   - Labels are generated for jumps (e.g., `lab = genlabel(2)` creates two labels: one for the if-true branch, one for else/skip)
   - Branches (`branch(label)`) emit jump code objects
   - `walk()` is called to emit expression evaluation code before conditionals
3. **Output:** 
   - Code object chain updated (`codelist` now points to newest code)
   - Statement labels registered in `stmtlabs` table (for goto resolution)
   - Function symbol updated with return label (`cfunc->u.f.label`) on RETURN

**Concrete example (if statement):**
```
if (cond) { stmt1 } else { stmt2 }
        ↓
walk(cond, 0, lab)     // evaluate condition, jump to lab if false
statement(...)         // emit stmt1
branch(lab+1)          // jump over else to lab+1
definelab(lab)         // else: label
statement(...)         // emit stmt2
definelab(lab+1)       // skip: label
```

## Learning Notes

- **Idiomatic LCC patterns:** This is a **three-phase compiler** (lex → parse/codegen → backend); statement parsing is phase 2. The code object chain is the IR; it's intentionally minimal (no SSA, no CFG optimization) because the backend will further lower to VM bytecode. A modern engine (Rust wasm, LLVM IR) would use structured CFG nodes or SSA form instead.

- **Game engine context:** The generated code objects eventually become **QVM bytecode** that Quake III's VM interpreter (`code/qcommon/vm_interpreted.c`) executes. Switch tables (`swcode()`) are particularly relevant because Quake III uses dense case-value packing for bot AI state machines (`code/game/ai_dmnet.c`), and this optimization directly impacts instruction cache behavior at runtime.

- **No VM-specific semantics here:** The statement parser is generic C; it has no knowledge of Quake III's game model (entities, trace syscalls, etc.). That semantics lives in `code/game/*.c` at the source level and is mediated by `trap_*` syscall wrappers at runtime. This separation keeps the compiler reusable.

- **Label management:** Contrast with modern compilers: LCC uses a simple counter (`genlabel()`) and linear search for label definitions (`findlabel()`). On modern backends (LLVM), basic blocks are structurally explicit. Here, labels are implicit in the code chain, which is simpler but less amenable to optimization passes.

## Potential Issues

- **No unreachable code elimination:** Warnings are issued but dead code is emitted anyway. A post-pass could strip unreachable sequences before codegen.
- **Switch table size:** A warning is issued at case count > 257, and for huge sparse ranges (> 10,000 values), but there's no hard limit. Pathological switches could exhaust hunk memory during compilation.
- **Label collision risk (low):** `genlabel()` uses a simple counter; no per-function reset, so label IDs are globally unique. This is safe but wastes ID space in large compilations.
