# lcc/src/c.h — Enhanced Analysis

## Architectural Role

This header is the **central compiler IR schema and backend interface contract** for the LCC compiler. It defines all core intermediate representation types (`Node`, `Type`, `Symbol`, `Tree`), the pluggable `Interface` vtable that backend implementations must populate, and pervasive macros for type-system queries and bit-level opcode manipulation. As the sole header included by nearly all LCC compilation units, it is the architectural glue binding the compiler frontend (lexing, parsing, semantic analysis in `decl.c`, `expr.c`, `stmt.c`) to multiple swappable backend generators (`alpha.md`, `x86.md`, `mips.md`, etc.).

## Key Cross-References

### Incoming (depends on this header)
- **Frontend**: `lex.c`, `expr.c`, `decl.c`, `stmt.c`, `gen.c`, `enode.c`, `tree.c`, `bind.c`, `types.c`, `sym.c`, `dag.c` — all parse/analyze into `Tree` and `Symbol` structures
- **Backends**: Each `.md` (machine description) file and corresponding output module (e.g., `x86.md` paired with code generation) consumes the `Interface` vtable to implement architecture-specific code emission
- **Main driver**: `main.c` links backends by calling `main()` and locating the correct `Binding` entry in the global `bindings[]` array to select a backend via platform/target
- **Output stage**: `output.c`, `pass2.c` consume populated `Interface` function pointers to emit final code

### Outgoing (what this file depends on)
- **`config.h`** (included; platform-specific): Sets `Metrics` for char/int/long/pointer sizes; configures target alignment rules
- **`token.h`** (included): Provides token enum definitions (`ID`, `CONST`, etc.) used in the `kind[]` extern array for quick token classification
- **Implicit dependency on all `.md` backend machine-description files** that populate the global `bindings[]` array; no direct C symbol references, but the linker resolves them

## Design Patterns & Rationale

1. **Pluggable Backend via Function Pointers (`Interface` vtable)**
   - Each backend exports `Binding bindings[] = { { "x86", &x86_interface }, ... }` with a fully populated `Interface` struct
   - The driver calls the backend's `progbeg()`, `function()`, `emit()`, `progend()` in sequence
   - **Rationale**: Late-bind target architecture; single source compiles to multiple ISAs without conditional compilation

2. **Bit-Packed Opcode Encoding**
   - Macros like `generic(op)`, `specific(op)`, `opindex()`, `opkind()`, `opsize()`, `optype()` extract overlapping bit ranges
   - Example: `specific(op) = (op)&0x3FF` extracts low 10 bits (base op + type); `opsize(op) = (op)>>10` extracts bits 10+
   - **Rationale**: Compact representation when storing thousands of IR nodes; minimizes memory footprint on 1990s machines

3. **Type System with Qualifier Wrapping**
   - `Type` nodes form a tree: `CONST` and `VOLATILE` nodes wrap a `type` pointer (qualifiers are separate nodes, not flags)
   - Macros `isqual()`, `unqual()`, `isconst()`, `isvolatile()` navigate this structure
   - **Rationale**: Correct C99 const/volatile semantics; preserves type identity across qualification layers for strict alias checking

4. **Dual Zone/Hunk Memory Model** (via extern declarations)
   - References to memory allocator in `qcommon` (game engine context), but LCC itself likely uses simpler zone via `NEW`/`NEW0` macros
   - **Rationale**: Fast fixed-size allocations with bulk deallocation; supports symbol table cleanup per scope

## Data Flow Through This File

1. **Parsing Phase** (not in this file, but produces structures defined here)
   - Tokens from `token.h` → consumed by `lex.c` → parsed by `expr.c`, `decl.c`, `stmt.c`
   - Results: `Tree` and `Symbol` objects allocated via `NEW()` macro

2. **IR Construction & Optimization**
   - `dag.c` builds DAG of `Node` trees using `kids[2]`, `link`, `syms[3]` fields
   - Type checking via predicates: `isint(t)`, `isarray(t)`, `isstruct(t)`, etc.
   - Opcode tagging via `mkop(op, ty)` macro: wraps generic op with type-specific suffix

3. **Code Generation**
   - `gen.c` walks IR tree, calls backend `Interface.emit(node)` for each node
   - Backend uses `node.op` (decoded via `generic/specific` macros) to dispatch on instruction type
   - Emits sequences of backend-specific instructions

4. **Symbol Tracking**
   - `sym.c` builds hash table of `Symbol` entries (lexical scope: `GLOBAL`, `PARAM`, `LOCAL`, `CONSTANTS`, `LABELS`)
   - `Symbol.uses` list tracks all references for later analysis (dead-code elimination, register allocation hints)

## Learning Notes

- **Pre-C99 Compiler Design**: This is architecturally identical to the Dragon Book (Aho/Sethi/Ullman), circa 1986. Shows how real production compilers (lcc is used in production) depart minimally from textbook IR and code-gen patterns.
- **Opcode Encoding**: The bitfield tricks (shifting op by 4 to embed type in low bits) is a memory optimization no longer necessary on modern systems; demonstrates how 1990s code squeezed resources.
- **Multi-Backend Strategy**: Unlike GCC (which hard-codes each backend), lcc's `bindings[]` array and `Interface` vtable show a cleaner plugin architecture that was ahead of its time. Modern LLVM adopted this philosophy.
- **Type Qualifiers as Wrapper Nodes**: The `CONST`, `VOLATILE` → `type` chain is not how modern compilers work (they use flag bits); this reflects stricter C89 semantics.
- **Integration with Quake III**: The presence of Quake III cross-references (game VM compilation) shows lcc was chosen for lightweight QVM bytecode compilation; its simplicity and small footprint made it ideal for embedded use.

## Potential Issues

1. **Bit-Packing Fragility**
   - The opcode macros assume specific bit layouts (`generic(op)=op&0x3F0`, `specific(op)=op&0x3FF`).
   - If a new opcode family exceeds 64 entries or size field exceeds 6 bits, unpacking silently produces wrong values.
   - No compile-time assertions; risk of misalignment if anyone adds opcodes without updating macro ranges.

2. **Global `Interface` Pointer**
   - `extern Interface *IR` is a global singleton; only one backend active at runtime.
   - Linking multiple backends simultaneously would be difficult (no explicit namespace isolation in C).

3. **Symbol Table Scope Management**
   - `Symbol.scope` is an int; no bounds checking on scope depth.
   - Deep lexical nesting could theoretically overflow, though K&R C lints typically warn at ~127 levels.

4. **No Built-In Overflow Protection for Opcode Fields**
   - `sizeop(n)` macro (`(n)<<10`) can silently overflow if `n >= 4096`, corrupting adjacent fields.
   - Should be replaced with safer multiply-and-check, but wasn't in 1990s code.
