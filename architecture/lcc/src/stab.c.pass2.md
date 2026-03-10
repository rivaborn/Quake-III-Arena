# lcc/src/stab.c — Enhanced Analysis

## Architectural Role

This file implements the STAB/DBX debugging symbol table generation backend for the LCC compiler. It operates as a code-generation phase that emits `.stabs` pseudo-instructions encoding source-level type definitions, variable scopes, function boundaries, and source line information. The module bridges the compiler's type system and symbol table (populated during semantic analysis) to a portable debugging format consumable by Unix/Linux debuggers, making it essential for generating debug-instrumented QVM bytecode.

## Key Cross-References

### Incoming (who depends on this file)

- **lcc main driver** (`lcc/src/main.c`, `pass2.c`): Calls `stabinit()` at compilation start, `stabline()` for each source location, `stabsym()` for each symbol, `stabtype()` for type definitions, `stabblock()` for scope entry/exit, and `stabend()` at EOF
- **Symbol table iteration** (`lcc/src/sym.c`): The `foreach(types, GLOBAL, ...)` call in `stabinit()` walks the global type registry maintained by the symbol/type system
- **Global type primitives**: `inttype`, `chartype`, `doubletype`, `floattype`, etc. from `lcc/src/types.c` are explicitly processed in `stabinit()` to emit baseline type codes

### Outgoing (what this file depends on)

- **IR abstraction** (`lcc/src/*md` files, e.g., `x86.md`, `sparc.md`): Queries `IR == &sparcIR` to emit IR-specific addressing modes; calls `IR->segment(CODE)` to emit label prefixes; calls `genlabel()` to allocate fresh debug labels
- **Type/symbol system** (`lcc/src/types.c`, `lcc/src/sym.c`): Reads type structures (`ty->op`, `ty->x.typeno`, `ty->u.sym`, fieldlists) and symbol metadata (`p->type`, `p->sclass`, `p->scope`, `p->x.regnode`)
- **Output stream** (`lcc/src/output.c`?): Calls global `print()` to write `.stabs` directives to the assembler input
- **Label generation** (`lcc/src/*`): Calls `genlabel()` and references current function via `cfunc->x.name`

## Design Patterns & Rationale

### Recursive Type Emission
`emittype()` is recursive, walking nested type structures (pointers, arrays, function signatures, struct fields). This mirrors how the type system itself is nested — a pointer to a struct requires emitting the pointed-to struct's definition first. The pattern avoids duplicate emission via the `ty->x.printed` flag, a simple memoization strategy fitting the compiler's arena-allocation era.

### Type Code Assignment & Memoization
`asgncode()` pre-assigns unique numeric codes (`ty->x.typeno`) to all types and marks them (`ty->x.marked`) to avoid re-traversal. This decouples the cost of generating the type code assignment from the cost of emitting the `.stabs` strings, allowing the compiler to defer string generation until needed.

### IR-Specific Branching
The checks `if (IR == &sparcIR)` produce two different output formats:
- **SPARC**: Uses `.stabd` (absolute line/block numbers, no address reference)
- **x86/others**: Uses `.stabn` with label arithmetic (`%s%d-%s` format), computing relative offsets to support per-function relocation

This hints at portability constraints: SPARC debug stabs could use absolute addressing; x86 required relative offsets for position-independent debug info.

### Scope-Driven Symbol Classification
`stabsym()` emits different STAB codes based on storage class and scope:
- `N_GSYM` (global), `N_LSYM` (local), `N_PSYM` (parameter), `N_RSYM` (register), `N_STSYM`/`N_LCSYM` (static)

This encodes scope rules in the debug output, allowing debuggers to validate variable visibility and lifetime.

## Data Flow Through This File

**Inbound:**
1. **Compilation phase sequence**: `stabinit()` → per-line `stabline()` → per-symbol `stabsym()`/per-type `stabtype()` → per-scope `stabblock()` → `stabend()`
2. **Type and symbol metadata**: Populated by earlier phases (lexing, parsing, semantic analysis); this phase reads only
3. **Global type registry**: Collected into `types` global by symbol table; `stabinit()` iterates it via `foreach()`

**Transformation:**
- Type tree → flattened numeric type codes (with recursive nested-type emission)
- Symbol records → `.stabs` directives with type codes, storage class indicators, location descriptors
- Source locations → `.stabs` line directives (using label offsets on non-SPARC)

**Outbound:**
- `.stabs` assembler pseudo-instructions emitted via `print()` to the compiler's output stream
- Consumed by the assembler and linker to generate `.stab`/`.stabstr` sections in the final object/executable

## Learning Notes

### Idiomatic to This Era (1990s–2000s Compilers)
- **Manual type numbering**: Modern compilers (LLVM, GCC post–5.0) use standardized DWARF format with automatic ID generation. Here, type codes are manually assigned and tracked, limiting portability.
- **String-based emission**: Debug info is emitted as assembly strings (`.stabs "..."`) rather than structured binary sections. This required the assembler to parse and re-encode — inefficient but simple to debug.
- **Single-pass symbol walk**: No separate debug-info tree; symbol table is walked in-place during code generation, relying on the IR to provide correct label references.
- **Scope via block markers**: Block scope is tracked via `N_LBRAC`/`N_RBRAC` stabs with nesting depth, rather than frame descriptor tables (DWARF3+).

### Contrast with Modern Engines
- **Game engines** (UE5, Unity) emit DWARF or similar to native debuggers; Quake 3's QVM uses minimal debug support (only for development builds)
- **IR design**: Modern compilers separate debug-info generation from code generation (e.g., LLVM has `DIBuilder`); here it's interleaved
- **Type system**: No composite type representation (no DW_TAG_typedef, DW_TAG_subroutine_type); types are flattened into a single namespace

### Connection to Quake 3 QVM Pipeline
This file is **never used at runtime** — it's only invoked when compiling C source files to QVM bytecode with debug symbols. The Quake 3 server/client binaries (game.qvm, cgame.qvm) would be built with this enabled; shipped binaries would likely be stripped to save space.

## Potential Issues

1. **No overflow checks on type numbering**: `ntypes` is a plain `int`; if a map has >2 billion unique types, it wraps. (Infeasible in practice, but not defensive.)
2. **Buffer size assumptions**: `stabinit()` allocates a 1024-byte stack buffer (`buf`) for `getcwd()`, which could overflow on deeply nested mount paths.
3. **Potential double-emission of struct/union types**: The logic at `emittype()` for STRUCT/UNION branches (checking `if (lev > 0 && (*ty->u.sym->name < '0' ...))`) skips emission for nested named types, but the memoization logic (`ty->x.printed`) may not prevent all duplicates if the traversal order varies.
4. **No null-safety on `cfunc->x.name`**: References to `cfunc` assume a current function context; if called outside a function scope, this dereferences a dangling pointer.
