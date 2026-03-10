# lcc/src/2html.c — Enhanced Analysis

## Architectural Role

This file is a **compiler IR visualization utility** within the LCC compiler toolchain (self-contained C compiler used to compile Quake III's QVM bytecode). It generates HTML documentation of the compiler's internal representation (RCC IR structures), bridging the gap between compile-time tooling and human-readable debugging output. While not part of the runtime engine, it plays a supporting role in the offline **Q3VM compilation pipeline**: lcc parses and optimizes C code, produces RCC IR trees (types, symbols, expressions, control flow), and `2html.c` helps developers inspect and validate these structures during compiler development and debugging.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC build system** (implicit): `2html.c` is compiled into the LCC compiler but only invoked through command-line flags or internal compiler hooks
- **No runtime engine dependencies**: This is a build-tool-only module; the game engine never calls this code
- Likely invoked by compiler introspection / debug output subsystems (not visible in provided source, but suggested by the `HTML` naming convention)

### Outgoing (what this file depends on)
- **RCC compiler headers** (`rcc.h`, `c.h`): Defines all IR structures this file visualizes
  - `rcc_type_ty`, `rcc_symbol_ty`, `rcc_item_ty`, `rcc_node_ty`, `rcc_interface_ty`, `rcc_program_ty` enum discriminated unions
  - Opcode constants (`CODE`, `BSS`, `DATA`, `LABEL`, `ASGN`, `CVT`, etc.)
  - Scope and storage class enums (`CONSTANTS`, `GLOBAL`, `PARAM`, `LOCAL`, `EXTERN`, `STATIC`)
- **Sequence abstraction** (`Seq_length`, `Seq_get`): Generic list/array utilities for iterating IR collections
- **Standard C library** (`stdio.h`, `stdlib.h`, `string.h`, `time.h`): File I/O and string manipulation

## Design Patterns & Rationale

### Macro-Heavy HTML Generation
- **Repetitive `do_*` functions**: Each IR type (type, symbol, node, interface) has its own output function that switches on a discriminant (`kind`) and outputs struct fields
- **`xx(ptr, field, type)` macro**: Reduces boilerplate for field printing: `xx(x, id, identifier)` expands to `printf("<li>id = "); do_identifier(x->id); printf("</li>\n")`
- **`yy(kind, field, type)` macro**: Variant access within switch cases; unifies the pattern of printing fields from variant union members (`x->v.rcc_KIND.field`)
- **`caselabel` macro**: Standardizes case label output with kind name and type annotation as HTML headings
- **Rationale**: Minimizes copy-paste errors in HTML structure; macro abuse is preferable to hand-written printf chains for dozens of similar fields

### Functional Dispatch on IR Kind
- Every IR node is a **discriminated union** with a `kind` field selecting one of several variant structures
- `do_*` functions use switch statements to handle each variant independently
- Enables layered output: `do_item()` → `do_type()` or `do_symbol()` → field-specific handlers
- **No object-oriented abstraction**: Functions are purely procedural; mirrors LCC's C-only design philosophy

### Direct HTML Emission (No Templating)
- All HTML is hard-coded `printf` statements; no template engine or buffer accumulation
- Generates **structured lists** (`<ul>`, `<ol>`, `<li>`) with semantic HTML5 anchor links (`<a href='#uid%d'>`)
- **Idempotency**: Each call to `do_program()` produces a complete, self-contained HTML fragment
- **Rationale**: Simplicity for a debugging/introspection tool; avoids external dependencies

## Data Flow Through This File

1. **Input**: `rcc_program_ty` (entire compiler IR tree) → passed to `do_program()`
2. **Processing**: 
   - Traverses the IR recursively via switch dispatch on `kind` fields
   - Calls type-specific handlers (`do_type`, `do_node`, `do_interface`) 
   - Emits HTML anchors for entity definitions (e.g., `do_define_uid` for symbols) and hyperlinks for references (e.g., `do_uid` for symbol references)
3. **Output**: Stream of HTML to `stdout` (via repeated `printf` calls)
4. **Key State**: `nextid` global counter (unused in visible code, possibly for ID generation)

## Learning Notes

### Compiler IR Representation Patterns
- **Discriminated unions** as the primary abstraction for IR nodes (type-safe variant storage without C's native tagged unions)
- **Sequence library** for homogeneous collections (generic container without C++ templates)
- **Scope and storage class enums** baked into symbol records — reflects Q3 game VM's need to distinguish local/global/parameter entities during compilation

### Design Philosophy
- **Minimal abstraction**: No visitor pattern, no IR walkers, just case statements and function pointers
- **Debug-friendly**: HTML output with cross-linked symbols is more readable than binary or text dumps
- **Self-documenting**: The enum→string mappings (e.g., `opnames[]`, `do_scope()`) serve as an informal IR reference

### Idiomatic to Pre-C99 Era
- No dynamic arrays or standard containers; Seq library provides basic list support
- Heavy macro use to simulate generic programming (pre-template C)
- Direct HTML generation reflects late-1990s/early-2000s tooling when templating engines were less common

## Potential Issues

1. **Buffer Overflows in String Tables**: `opnames[]` and `suffixes[]` arrays are fixed-size. If opcodes are added to RCC beyond the 46 entries in `opnames`, `do_op()` will silently fall back to printing the raw integer. The `suffixes[]` array (10 chars) similarly doesn't validate bounds.

2. **Hyperlink Bugs**: `do_uid()` and `do_label()` format hyperlinks as `<a href='#%d'>%d</a>` and `<a href='#L%d'>%d</a>`, but print three arguments (`%d, %d, %d`) in the first — a **format string mismatch** that will cause undefined behavior or corrupted output.

3. **Incomplete Variable Initialization**: Line 171 declares `int i` twice in `do_uid_list()` — the second declaration shadows the first, though both are valid C89 (likely harmless but sloppy).

4. **Missing Validation**: No error handling if IR structures are malformed or null; assumes all linked IR data is well-formed.

5. **Truncated Output in First-Pass Doc**: The source file is 555 lines, but the provided content is truncated at ~500 lines, so the `do_program()` function and module-level entry points are not visible in this analysis.

---

**Note**: This file does **not** interact with the runtime engine, game VMs, or networking code. It is pure offline tooling for LCC compiler development and IR inspection. Its role in the Q3 build system is to provide human-readable diagnostics during compilation, not to affect game behavior.
