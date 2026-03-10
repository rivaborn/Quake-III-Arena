# lcc/src/dag.c — Enhanced Analysis

## Architectural Role

This file implements the **Intermediate Code Generation** phase of the LCC C compiler, converting an abstract syntax tree (AST) into a directed acyclic graph (DAG) representation. It serves as the critical bridge between the parser and the platform-specific code generators in the backend. The DAG form enables common sub-expression elimination (CSE) and prepares code for register allocation by normalizing expression trees into a canonical form that deduplicates equivalent subexpressions and sequences operations as a linked forest.

## Key Cross-References

### Incoming (who calls this file)
- **`common.c` / compiler driver**: calls `walk()` as the entry point to convert parsed statements into IR
- **Compiler's code generation infrastructure**: invokes `gencode()` and `emitcode()` to lower DAGs to target-specific code
- **Parser frontend** (`expr.c`, `stmt.c`): supplies `Tree` AST nodes that are recursively processed by `listnodes()`

### Outgoing (what this file depends on)
- **`c.h`**: compiler header exposing `Node`, `Tree`, `Symbol`, `Code` structures; operation macros (`generic()`, `opindex()`, `sizeop()`)
- **IR backend** (`code/qcommon/vm.c` style code generation): consumes the `forest` linked list via `code(Gen)->u.forest`
- **Memory infrastructure** (`allocate()`, `deallocate()` via `c.h`): manages DAG node allocation in statement-scoped pools
- **Symbol table** (via `Symbol` references): tracks variable identities and temporary names for code generation

## Design Patterns & Rationale

**Hash-table DAG canonicalization**: The `buckets[16]` array (line 9) implements a fixed-size hash table for deduplication. When `listnodes()` builds nodes via `node()`, it first checks if an equivalent node already exists (same operation, operands, symbol). This CSE approach reduces code size and improves efficiency by eliminating redundant calculations — a classic compiler optimization, especially valuable for a resource-constrained QVM bytecode target.

**Depth-first tree traversal with control-flow labels**: `listnodes()` is a massive switch statement (lines 102–577) that recursively processes each AST node type. Special handling for `AND`/`OR`/`NOT`/`COND` maintains label depth (`depth++`/`depth--`) and issues jump instructions, ensuring precise control-flow semantics without explicit CFG construction — a pragmatic choice for a simple compiler targeting label-based IR.

**Temporary node pools and `forest` linked list**: The `forest` global (line 7) is a circularly linked list of DAG nodes representing sequenced statements. The `list()` function inserts nodes, and `reset()` clears stale memoization between statements, enforcing statement-level independence — critical for deterministic code generation and debugging.

**Volatile/struct field special casing**: Lines 445–490 (INDIR, FIELD cases) show that the code generator avoids CSE for volatile-qualified data and field extractions, forcing fresh `newnode()` allocations instead of cached `node()` calls. This reflects the need to preserve side effects and memory access semantics.

## Data Flow Through This File

1. **Input**: `Tree tp` (AST node from parser) + control-flow labels (`tlab`, `flab`)
2. **Traversal**: `listnodes()` recursively walks children, issuing intermediate operations
3. **Canonicalization**: Equivalent subexpressions are deduplicated via the hash table; `node()` returns cached nodes
4. **Sequencing**: Nodes are linked into `forest`; statements are separated by `reset()`
5. **Output**: A `forest` of DAG nodes is passed to platform-specific code generation via `code(Gen)->u.forest`
6. **Post-processing**: `gencode()` and `emitcode()` apply backend-specific pruning (`prune()`), fixup, and code emission

The flow respects C's evaluation order and side-effect semantics: assignments kill cached nodes (`kill()`, line 77), volatile/field accesses bypass caching, and control flow is made explicit via labels.

## Learning Notes

**Idiomatic to era (1990s–2000s compiler design)**: This file exemplifies pre-SSA intermediate representations. Modern compilers use explicit SSA (Static Single Assignment) or sea-of-nodes graphs, whereas `dag.c` relies on imperative value numbering and statement-level resets. The lack of an explicit CFG is compensated by inline label threading.

**CSE via memoization is simple but limited**: The fixed 16-bucket hash table is deliberately small; collisions are cheap because nodes with identical ops/operands are semantically equivalent. However, this approach misses CSE opportunities across statement boundaries (since `reset()` clears buckets between statements).

**Label-based control flow reflects the QVM target**: The frequent use of `genlabel()`, `findlabel()`, and `jump()` operations show that the backend is label-oriented (QVM and Quake's native VMs are stack machines with explicit labels). Modern register-allocating compilers would use a CFG with blocks, but this is simpler and maps naturally to the target.

**The `iscall()` macro and multiplication-as-call**: Lines 3–5 show that division, modulo, and multiplication can be treated as function calls on certain platforms (`IR->mulops_calls`). This exposes a key design point: the IR is platform-parametrized via `IR` globals, allowing the same front-end to target diverse backends.

**Shared field and bit manipulation logic**: The ASGN case (lines 402–438) contains complex bit-field assignment logic (shifting, masking, ORing), showing that the front-end is not always generating the most canonical form — the DAG builder must handle the semantic complexity of C's field extraction.

## Potential Issues

**No explicit dead-code elimination**: Unreachable nodes may linger in the DAG if control flow is mispredicted or labels are equated. The `reset()` mechanism assumes statement-level cleanup is sufficient, but cross-statement garbage is not collected.

**Fixed hash table size**: The 16-bucket hash table is a single magic constant with no scaling. High-temp or deeply-nested code could suffer collision chains. The `prunetemps` flag (line 13) suggests awareness of temp bloat, but the hash table itself is not resized.

**Label equating not validated globally**: The `equated()` function (lines 631–634) follows label chains at code-emit time but does not pre-validate consistency. Circular label equating is prevented by assertion, but the mechanism is reactive rather than proactive.
