# lcc/src/token.h — Enhanced Analysis

## Architectural Role

This file is the **canonical token registry** for the LCC C compiler used by Quake III's build pipeline. It defines all lexical tokens (keywords, operators, literals, punctuation) that the compiler's lexer recognizes and passes to the parser. The file serves as a single source of truth: by using `xx`/`yy` macro indirection, it enables multiple compilation phases (lexer symbol tables, parser precedence tables, AST node factories) to consume the same token definitions without duplication or drift. This is essential infrastructure for the offline QVM bytecode compilation toolchain (game, cgame, ui VMs).

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/src/lex.c`**: Lexer uses token definitions to initialize symbol tables and recognize keywords during tokenization
- **`lcc/src/parse.c` / `lcc/src/expr.c`**: Parser and expression evaluator consume token precedence values and operator codes
- **`lcc/src/dag.c`**: DAG builder uses operator codes (ADD, SUB, MUL, etc.) to construct intermediate representation nodes
- **`lcc/src/gen.c` / code generation phases**: Use operator codes to dispatch to backend code generators
- Build scripts (`.mak`, `Conscript`): Indirectly via compiler binary that embeds these definitions

### Outgoing (what this file depends on)
- **No runtime dependencies**: This is a pure definition file; it only exports symbol names and constants
- Macro definitions (`xx`, `yy`) are expected to be defined by the **including file** before `#include "token.h"`
- Conceptually relates to C language standards (C89/C99 keyword set, operator precedence per C spec)

## Design Patterns & Rationale

### Token Registry as Macro-Expandable Definition
The file uses a **preprocessor-driven code generation pattern**:
```c
xx(INT, 5, 0, 0, 0, CHAR, "int")
yy(0,   17, 0, 0, 0, 0, 0)
```
Different including files define `xx`/`yy` macros to extract different views:
- **Lexer** (`lex.c`): `#define xx(name, val, ...) { name, val, # name }` → builds keyword lookup table
- **Parser** (`expr.c`): `#define xx(name, val, prec, ...) { name, prec, ... }` → builds precedence table
- **Code gen**: `#define xx(..., op, ...) { op, handler_fn }` → builds operator dispatch table

This avoids the maintenance burden of keeping three separate tables synchronized.

### Sparse Token Space with Reserved Slots
- Values 0–31: Type keywords, special tokens, and reserved slots (17–30 unused, 28 and 31 reused for "long long" and "const volatile")
- Values 32–36: Identifiers and literal constants
- Values 37–63: Operators and punctuation (with gaps)
- Values 64–126: Control-flow keywords and misc operators
- Value 127: `EOI` (end of input) sentinel

This layout suggests the token space was designed incrementally; unused slots may represent removed tokens or room for future extensions.

### Operator Metadata Encoding
Each operator token carries:
- **Precedence** (column 3): Controls expression parsing order (lower precedence = looser binding)
  - Precedence 2: assignment (`=`)
  - Precedence 5: logical AND (`&&`)
  - Precedence 13: multiplication/division (`*`, `/`, `%`)
- **Operator code** (column 4, e.g., `ADD`, `MUL`, `EQ`): Used in DAG construction and backend dispatch
- **Tree type** (column 5, e.g., `addtree`, `eqtree`): Hints for code generation strategy (arithmetic vs. comparison)
- **Kind** (column 6, e.g., `ID`, `CHAR`): Semantic category used by parser for type checking and error messages

### Why `xx` and `yy` Macros?
- `xx`: "defined token" — has a meaningful definition, contributes to the language grammar
- `yy`: "reserved/stub token" — occupies a token ID slot but carries no semantic payload (often all zeros)

This naming convention likely comes from yacc/Bison generator conventions and signals "production" vs. "non-production" entries.

## Data Flow Through This File

**Input**: C language specification (keyword set, operator precedence, operator semantics per C89/C99 standard)

**Transformation**: Static text file with columnar macro-invocation syntax

**Output**: 
1. **Lexer symbol tables** (via `lex.c` #include): Maps keyword strings → token IDs for fast keyword recognition
2. **Parser precedence/associativity tables** (via `expr.c` #include): Enables Pratt-style operator precedence parsing
3. **Backend dispatch tables** (via `dag.c`/`gen.c`): Routes operators to code generation routines
4. **Error messages**: Token string representations used in diagnostic output

No runtime state mutations; purely declarative. The file is included at **compile-time only** during the LCC build; the resulting compiled `lcc` binary embeds all token definitions in its data sections.

## Learning Notes

### Idiomatic Compiler Design Pattern
This approach (macro-driven definition tables) was common in compilers written in the 1980s–2000s (cf. Kernighan & Ritchie's "The C Programming Language," traditional yacc/Bison workflows). Modern compilers often use:
- Codegen scripts (Perl/Python) to emit multiple language-specific files
- DSL-based token specifications (ANTLR, tree-sitter)
- Data-driven metadata (JSON/YAML schemas)

But the macro approach is **minimal and self-contained**: no build dependencies, no code generation step, single file of truth.

### Token Value Numbering Reflects Language Layers
- **0–16**: Type system (fundamental types from C spec)
- **17–31**: Extension/reserved slots (compiler-specific or future features, e.g., "long long" at 28)
- **32+**: Identifiers, constants, operators, keywords (runtime categories, not types)

This mirrors how a recursive-descent or table-driven parser would group tokens by grammatical role.

### Precedence Values Map to C Operator Associativity
Columns 3–5 encode the **Pratt binding power** and **DAG tree type**:
- Assignment (prec 2, `ASGN`, `asgntree`) is right-associative and lowest precedence
- Logical AND/OR (prec 4–5) are left-associative with medium precedence
- Arithmetic/bitwise (prec 7–13) follow standard C precedence

A parser can use these values to implement **operator-precedence climbing** without hard-coded grammar rules per operator.

## Potential Issues

1. **Token Space Fragmentation**: Values 17–30 and gaps in 64–126 are unused, making the token space somewhat wasteful and confusing for maintainers. A comment map would help.

2. **Operator Tree Type Semantics Undocumented**: Column 5 values like `addtree`, `cmptree`, `bittree` are not self-documenting. Without reading `gen.c`, it's unclear what backend behavior each triggers.

3. **"Kind" Column Overloading**: Column 6 uses both token IDs (e.g., `ID`, `CHAR`, `IF`) and literal characters (e.g., `')'`, `','`, `'^'`) inconsistently. This works but reduces clarity; could be normalized to always use token IDs or always use strings.

4. **Macro Fragility**: If an including file forgets to `#undef xx` / `#undef yy` after including this file, subsequent includes may misbehave. The file does provide `#undef xx` and `#undef yy` at the end, which is good hygiene, but the pattern is error-prone.

5. **No Version Control**: The token set is frozen at C89 + some Q3-specific extensions (e.g., `__typecode`, `__firstarg`). Adding modern C features (C99 `restrict`, C11 `_Alignof`, etc.) would require careful coordination across multiple backend files and is not reflected in any feature flag or version enum.
