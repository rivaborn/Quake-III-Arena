# lcc/cpp/cpp.c — Enhanced Analysis

## Architectural Role

This file is the **C preprocessor front-end** in the LCC compiler toolchain—part of the offline build-time infrastructure, completely separate from the runtime engine. It is responsible for tokenizing Q3 game source code, expanding macros, and resolving conditional compilation directives before the parsed intermediate representation reaches the compiler proper. The output feeds directly into the parser phase (`lcc/src/input.c`, `lex.c`), making this a critical gate in the QVM bytecode generation pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/src/main.c`** calls `main()` after detecting preprocessor invocation (`-E` flag or implicit phase)
- **`lcc/cpp/`** lexer subsystem (`lex.c`, `macro.c`, `nlist.c`, `include.c`) consumes token definitions and macro table state maintained here
- **`lcc/src/input.c`** / **`lex.c`** downstream parser reads the output buffer (`outbuf`) via `puttokens()`

### Outgoing (what this file depends on)
- **`cpp.h`** — defines all public types (`Tokenrow`, `Token`, `Source`, `Nlist`)
- **Vendored utilities** (`lcc/cpp/lex.c`, `include.c`, `macro.c`, `nlist.c`) for tokenization, macro expansion, and conditional stack management
- **`lcc/etc/linux.c`** / **`lcc/etc/win32.c`** — platform-specific setup hooks (called indirectly via `setup()`)
- **Standard C library** — `<stdio.h>`, `<time.h>`, `<stdarg.h>` for I/O and variable-argument error reporting

## Design Patterns & Rationale

**Stateful Token Stream Processing:**
- `Tokenrow tr` is a reusable buffer (fixed capacity, sliding window pointers `bp`, `tp`, `lp`)
- Each pass through `process()` refills `tr` only when exhausted (`tp >= lp`), minimizing I/O overhead
- Token expansion and macro substitution happen in-place within the row, mutating `tp` to skip or inject tokens

**Nested Conditional Stack** (`ifdepth`, `ifsatisfied[]`, `skipping`):
- Rather than recursively parsing `#if` nesting, the preprocessor maintains a flat integer stack tracking depth and satisfaction state
- `skipping` acts as a bitmask: if it equals `ifdepth`, the current block is skipped; clever bit-packing avoids explicit stack allocation
- This is idiomatic for 1980s/90s C preprocessors before template metaprogramming made conditional semantics more complex

**Error Reporting via `va_list`:**
- All diagnostics converge through `error()` with format specifiers (`%s`, `%d`, `%t` for Token, `%r` for Tokenrow)
- Source location is printed by walking the `cursource` chain backwards to root, showing the include stack—essential for debugging long include chains
- `nerrs` is a simple counter; no structured error object needed in this era

**Manual Memory Management (era-appropriate):**
- `domalloc()` / `dofree()` wrap malloc/free with fatal OOM handling; consistent with LCC's overall design philosophy
- No use of `goto` for early returns; control flow is explicit and traceable

## Data Flow Through This File

```
Input:  Source file (via FS subsystem) → gettokens() fills Tokenrow
        ↓
Token Stream Processing:
  - Control directives (#define, #ifdef, #line, etc.)
    → control() dispatches to handlers (dodefine, doinclude, eval)
    → modifies macro table & conditional state
  - Regular tokens → expandrow() applies macros in place
        ↓
Output: Processed token stream → puttokens() → outbuf → downstream parser
```

**Key State Transitions:**
- `incdepth++` on `#include` (via `doinclude()`); `incdepth--` on END token from nested file
- `ifdepth++` on `#if`/`#ifdef`/`#ifndef`; conditional `skipping` toggled by `#elif`/`#else`; `ifdepth--` on `#endif`
- `cursource` chain: each `#include` pushes a new `Source` node; `unsetsource()` pops on END

## Learning Notes

**Idiomatic 1990s C Preprocessor Design:**
- Single-pass tokenization with lookahead (not AST-based); macro expansion happens lexically, not semantically
- No concept of "hygiene" (macro parameter capture can collide with caller identifiers)—this is pre-Scheme-macro-era thinking
- `#line` and `#error` are relatively late additions; the core design predates modern debugging needs

**Contrast with Modern Tools:**
- Modern preprocessors (LLVM Clang's `-E`, GCC's cpp) use a multi-pass AST approach with proper token provenance tracking
- The "skipping during parse" approach here is O(n) in nested depth; modern tools use an AST pass with O(1) branch elimination
- No macro argument hygiene; modern systems use token-tagging to prevent accidental capture

**Engine Connection:**
- Although this file is strictly offline/compile-time, **it enables the game's scriptability**: bot AI (`.ai` files), game logic, and UI scripting all pass through this preprocessor before QVM compilation
- Understanding this pipeline is critical to modifying Q3's bot scripting language or game rules—changes to conditional logic here ripple through all downstream game behavior

## Potential Issues

- **Include-guard pattern inefficiency:** The preprocessor re-expands macros on every `#include` pass. Modern tools use pragma-once or include-guard detection to skip re-tokenization.
- **No line number tracking for macro expansions:** Diagnostics from expanded macros report the expansion site, not the definition—makes debugging complex macros harder.
- **`ifsatisfied[ifdepth]` uses magic value 2 for "else seen":** A bitmask or enum would be clearer and safer against future modifications.
- **Unbounded `#include` nesting:** The `NIF` limit is checked, but stack exhaustion during extremely deep includes is possible if NIF is not sized conservatively.
