# lcc/src/decl.c — Enhanced Analysis

## Architectural Role

`decl.c` is the **declaration parsing and symbol management phase** of the LCC compiler, which compiles C code down to QVM (Quake Virtual Machine) bytecode. This is an **offline-only tool** used to pre-compile `code/game/`, `code/cgame/`, and `code/ui/` modules into QVM DLL/bytecode before deployment. It bridges the lexer (token stream) with the backend code generator via a symbol table and type system, handling all semantic rules for global/local scope, storage classes, and function/parameter declarations.

## Key Cross-References

### Incoming (Callers)
- `lcc/src/main.c` calls `program()` as the top-level parse entry point after lexing
- `lcc/src/stmt.c` and `lcc/src/expr.c` call `dclr()` / `dclr1()` for declarator parsing in initializer/cast contexts
- `lcc/src/enode.c`, `lcc/src/tree.c` reference symbols created by this module (via global `Symbol` handles)

### Outgoing (Dependencies)
- **Symbol table**: `lookup()`, `install()`, `relocate()` from unspecified symbol infrastructure (likely `lcc/src/sym.c`)
- **Memory**: `newarray()`, `ltov()`, `install()` allocate via zone allocator
- **Scoping**: `enterscope()`, `exitscope()` manage lexical scope depth (`level` global variable: GLOBAL=0, LOCAL, PARAM)
- **Type system**: `tsym` (current token symbol), `qual()`, `unqual()`, `ptr()`, `array()`, `func()` type constructors
- **Backend**: `IR->defsymbol()`, `IR->export()`, `IR->global()`, `IR->stabsym()` dispatch to code-gen layer
- **Error/diagnostic**: `error()`, `warning()`, `test()`, `skipto()`—lexical analysis is assumed to provide `t` (current token) and `token` (string value)

## Design Patterns & Rationale

1. **Recursive-Descent Declarator Parsing** (`dclr1()` builds a type skeleton as a linked list of operator nodes, then wraps the base type in `dclr()`). This mirrors the grammar: `*` (pointer) and `[]`/`()` (postfix) are applied inside-out, but the C syntax requires reading them left-to-right. The skeleton approach defers the wrapping until the base type is known.

2. **Scope-Driven Symbol Installation**: Uses a simple linear scope depth (`level`) and per-scope symbol lists (`identifiers`, `globals`, `externals`). No hash tables visible here—lookups delegate to the symbol module. This is idiomatic of 1990s compiler design.

3. **Type Composition & Qualifier Tracking**: The `specifier()` function carefully tracks `CONST` and `VOLATILE` orthogonally from base types, then applies them uniformly to the final type. The code handles edge cases (e.g., `const int*` vs `int * const`) via explicit `qual()` wrapping.

4. **Dual Symbol Instances for Forward Compatibility**: Global symbols may exist in both `externals` (forward declared) and `globals` (defined). The code uses `relocate()` to merge them—avoiding duplicate symbols while preserving declaration order.

5. **Old-style vs. ANSI Prototypes**: `parameters()` detects and flags old-style `int foo(a, b) {}` declarations (`fty->u.f.oldstyle = 1`) separately from ANSI `int foo(int a, int b) {}`. This reflects support for K&R C code still common in the late 1990s.

## Data Flow Through This File

1. **Lexical Entry**: `program()` loops while `t != EOI`, dispatching to `decl(dclglobal)` for global declarations.
2. **Specifier Parsing**: `specifier()` consumes type keyword tokens (int, float, const, etc.) and builds a base `Type` object.
3. **Declarator Parsing**: `dclr()` recursively processes `*`, `()`, `[]` syntax to construct the final type, storing the identifier in `id`.
4. **Symbol Installation**: 
   - Global: `dclglobal()` installs in `globals` or relocates from `externals`; checks for redeclaration conflicts; applies `IR->defsymbol()` / `IR->export()` callbacks.
   - Local/Param: `dcllocal()` and `dclparam()` install in `identifiers` with appropriate scope/lifetime.
5. **Backend Dispatch**: Calls like `defglobal(p, DATA)` and `IR->stabsym(p)` write stabs debug info or emit data sections.
6. **Scope Management**: `enterscope()` / `exitscope()` called around function parameters to enforce parameter-only visibility; `exitparams()` validates old-style parameter lists.

## Learning Notes

- **Compiler Phase Architecture**: This file exemplifies the separation between *syntax* (what the grammar allows) and *semantics* (scope, type checking, symbol lifetime). The recursive-descent structure is straightforward to follow, unlike table-driven LL(1) parsers.
- **Type Representation in C Compilers**: Modern compilers (LLVM, GCC) use rich intermediate representations (AST nodes with type info). LCC's approach—building a singly-linked type skeleton and wrapping the base—is memory-efficient but less flexible for complex type queries.
- **K&R Legacy**: The distinction between old-style and ANSI parameter lists highlights mid-1990s C compiler concerns; modern codebases have dropped K&R support entirely.
- **Symbol Table Discipline**: No explicit hash table here; the symbol module abstracts it away. Real compilers often use 2–3-level hierarchies (file scope, function scope, block scope) with scope-chain lookups—LCC's flat `level` counter is simpler.

## Potential Issues

- **No explicit handling of forward function declarations within a file**: `dclglobal()` will relocate externals→globals, but deeply nested scope dependencies (e.g., local struct member types) are not modeled; the code assumes the C grammar itself prevents issues.
- **Limited type-equivalence checking**: `eqtype()` is called for redeclaration checking, but the code doesn't deeply validate parameter list compatibility across function redeclarations.
- **Memory overhead**: Type skeletons (via `tnode()`) are allocated per declarator on each `NEW0(ty, STMT)`, even for simple cases. No pooling or interning.
