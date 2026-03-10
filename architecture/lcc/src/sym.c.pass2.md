Looking at `lcc/src/sym.c`, I'll provide an enhanced architectural analysis incorporating the cross-system context.

---

# lcc/src/sym.c — Enhanced Analysis

## Architectural Role

This file implements the symbol table for the **LCC C compiler**, a standalone tool (not part of the runtime engine) used to compile game code to QVM bytecode. The symbol table is purely a compile-time data structure that manages identifier visibility, scope levels, and constant interning during semantic analysis. The output of compilation using this symbol table becomes QVM bytecode later executed by `code/qcommon/vm_interpreted.c`, `vm_x86.c`, or `vm_ppc.c`, but the symbol table itself never exists at runtime.

## Key Cross-References

### Incoming (what calls sym.c)
- **LCC parser/semantic analyzer** (`lcc/src/decl.c`, `expr.c`, `stmt.c`, `sym.c` itself via recursive lookups)
- Symbol queries during type checking and code generation
- Scope entry/exit on function/block boundaries

### Outgoing (what sym.c depends on)
- **Memory allocation**: `NEW0` macro (from `lcc/src/`) allocates from persistent or function-local arenas
- **String interning**: uses `stringd()` and `string()` helpers (elsewhere in lcc)
- **VM backend interface**: `IR->defsymbol()` callback is invoked to notify the IR/code-gen backend about symbol definitions (e.g., for STATIC constants, GLOBAL idents)

## Design Patterns & Rationale

### 1. **Scope Chain with Hash-Table Chaining**
- Each scope level is a separate `struct table` with 256 hash buckets
- Tables form a linked list via `previous` pointer (scope stack)
- `lookup()` traverses the scope chain, finding closest enclosing definition
- **Rationale**: Efficient O(1) lookup in common case; handles shadowing naturally

### 2. **Parallel Symbol Tables by Category**
- Separate global tables for `constants`, `identifiers`, `externals`, `types`, `labels`
- **Rationale**: Type symbols have different lifecycle (must persist for struct/union/enum definitions across compilation units); constants are interned for sharing; labels are compiler-generated; externals track EXTERN linkage separately from local IDs
- Mirrors C's rule that `struct`, `union`, `enum`, and `typedef` names live in a separate namespace from ordinary identifiers

### 3. **Constant Interning (Deduplication)**
- `constant()` searches existing constants before creating new ones
- Uses value+type to determine uniqueness
- **Rationale**: Multiple identical constants (e.g., `"hello"` string literals, `42` in different expressions) share one symbol entry; reduces code size, enables folding

### 4. **Insertion-Order Traversal via `up` Field**
- All symbols in a table are singly-linked via `sym.up` in reverse insertion order
- Enables `foreach()` to visit scopes in declaration order (useful for debugging, error reporting)
- Hash table is for lookup only; traversal uses the `up` chain

### 5. **IR Backend Callback on Symbol Definition**
- `(*IR->defsymbol)(p)` is called for STATIC constants, GLOBAL idents, labels
- Decouples symbol table from code generation; backend can emit symbol records, allocate storage, etc.
- **Not used for local variables** (block scope) because those are allocated during code gen

## Data Flow Through This File

```
Parser/Semantic Analyzer
  → enterscope() / exitscope()     [manage C block/function scope stack]
  → install(name, &table, level)   [add new symbol to table at scope]
  → lookup(name, table)            [search symbol in scope chain]
  → constant(type, value)          [intern constant, call IR→defsymbol]
  → temporary() / genident()       [create compiler-generated temporaries]
  → vtoa(type, value)              [convert const value to string for output]
  ↓
Symbol Table State
  [constants hash table + linked list]
  [identifiers hash table + linked list]
  [types hash table + linked list]
  [labels hash table + linked list]
  ↓
Code Generator (via IR vtable)
  [emit symbol records, allocate storage]
```

## Learning Notes

### Scope Representation
- **Level numbers**: `GLOBAL` = 0, `LOCAL` starts at 1; each nested block/function increments
- Contrast with modern ECS/graph-based engines: C's linear scope stack is simple but rigid
- The `level >= LOCAL ? FUNC : PERM` arena choice in `genident()` shows: temporaries at function scope live on the function-local arena (freed after compilation); global temps live on permanent arena

### Constant String Values
- `mkstr()` creates an array-of-char constant for string literals; the `p->u.c.loc` field holds a generated STATIC identifier for the data itself
- String constants are never inlined into code; they become STATIC data with names like `_L10`

### Temporary vs. Identifier
- `temporary()` creates unnamed locals (named by incrementing `tempid`); e.g., `_1`, `_2`
- `genident()` creates named compiler-generated symbols (e.g., label strings from `genlabel()`)
- **Both** are marked `generated = 1` to distinguish from source-level identifiers

### Idiomatic C Compiler Pattern
- This is textbook **dragon-book scope handling**: hash tables + linear scope chain
- Modern languages often use AST-based symbol graphs or hierarchical environments, but procedural C compilers use this tabular style for cache locality and simplicity
- The `equalp(x)` macro leveraging a union of value types (`v.i`, `v.u`, `v.d`, `v.p`, `v.g`) is old-school but effective for interning without polymorphism

## Potential Issues

1. **Hash collisions on 256 buckets**: Linear chaining with 1024+ symbols could degrade to O(n); no rehashing. For QVM code this is acceptable (scripts are small), but large modules might see slowdown.

2. **Scope level integer overflow**: `level` is a plain `int`; deeply nested code could theoretically overflow. Not a practical Q3 issue given map limits, but not defended.

3. **No scope exit validation**: `exitscope()` doesn't check that the exiting scope is the innermost; callers must discipline themselves to match enter/exit. Contrast with stack-based allocation guards.

4. **Identifier warning threshold hardcoded**: `if (++n > 127)` in `exitscope()` checks only when `Aflag >= 2`. The 127 limit is arbitrary and not configurable.

5. **`IR->defsymbol()` side effects not isolated**: Calling IR backend from symbol table layer couples code generation to symbol lookup; if IR is stateful, unexpected mutation can occur.
