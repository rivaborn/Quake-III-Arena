# lcc/src/error.c — Enhanced Analysis

## Architectural Role

This file implements error handling for the **LCC C compiler**, an offline tool used to compile game VM source code (game, cgame, ui modules) into QVM bytecode. It is **not part of the runtime engine**—it's part of the build toolchain. The error management infrastructure here directly supports LCC's parser (`expr.c`, `stmt.c`, `decl.c`) and lexer (`lex.c`), providing formatted diagnostics with source location tracking, error recovery, and configurable error limits.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC parser/lexer stack** (`lex.c`, `expr.c`, `stmt.c`, `decl.c`, `input.c`): these consume `error()`, `expect()`, `test()` for diagnostic reporting and `skipto()` for error recovery
- **LCC main compilation loop** (`main.c`): respects the global `errcnt` to decide whether to emit object code
- All lcc source modules read global `wflag` to conditionally suppress warnings

### Outgoing (what this file depends on)
- **Global variables** from other lcc modules: `t` (current token), `token` (token text), `src` (source location record), `firstfile`, `cp`, `tsym` (symbol table entry for current token)
- **Custom stdio wrapper** (`fprint`, `vfprint`): likely defined in lcc's stdio abstraction layer (not shown, but referenced throughout lcc)
- **Token metadata** (`kind[]` array): generated from `token.h` via `#include` macro trick; maps token type → character class for `skipto()` set matching

## Design Patterns & Rationale

**1. Error Counting with Saturation**
- `errcnt` caps at `errlimit` (default 20), then sets to -1 sentinel, then exits. This prevents overwhelming users with cascading parse errors while allowing enough diagnostics to pinpoint root cause.
- Rationale: 1990s-era compiler pragmatism—bailing early keeps compiler predictable and prevents log spam.

**2. Error Recovery via Token Skipping**
- `skipto(tok, set[])` is a classic **panic-mode recovery**: after a syntax error, scan forward until finding a synchronization token (in `set`) or target token, printing skipped tokens for context.
- Character-based classification (`kind[t]`) allows recovery to group by semantic token families (e.g., "skip until any statement starter").

**3. Differentiated Warnings**
- `wflag` is a global suppress-warnings flag. When 0, `warning()` decrements `errcnt` (counterintuitively), then prints "warning: " prefix. This allows warnings to be accumulated without inflating the error count.

**4. Source Location Tracking**
- `src` record (struct not shown here) is printed on every error with `%w` format specifier. This ties errors to file:line context. Tracked across `firstfile` (initial file) and current `file` (via conditional in `error()`).

## Data Flow Through This File

```
Parser calls expect(tok)
  ├─ if (t == tok) consume via gettok()
  └─ else: error("syntax error; found")
           printtoken()
           fprint(...expecting...)
           
Later, skipto() is called to recover:
  └─ Loop: gettok() until t matches tok or char-kind is in set[]
           printtoken() on skipped tokens (first 8 shown, then "...")
           
warning() decrements errcnt to distinguish from hard errors
fatal() sets errcnt=-1 as checkpoint for exit
```

## Learning Notes

**Idiomatic to this era:**
- Macro-generated token classification (`#define xx / #include token.h`): avoids hand-maintained enum↔string mappings, but requires understanding the two-level macro trick.
- Panic-mode error recovery: minimally sophisticated by modern standards (no AST rollback, no context-aware prediction).
- Global state everywhere (`errcnt`, `wflag`, `t`, `token`, `src`): typical of 1980s–1990s compilers; modern compilers pass context structs.
- Custom `fprint` wrapper: suggests LCC may be ported across platforms with inconsistent stdio (note the `%w`, `%k`, `%S` custom format specifiers).

**Connection to Q3 architecture:**
- LCC is only invoked **offline** during game development to compile `.c` → `.qvm` bytecode.
- No runtime engine code calls into `lcc/` directly; LCC's output (bytecode) is consumed by `qcommon/vm_interpreted.c`, `qcommon/vm_x86.c`, etc., which **host** the compiled code.
- The error handling here is developer-facing (compile-time diagnostics), not player-facing.

## Potential Issues

1. **Unbounded string formatting** in `printtoken()`: loops print up to 20 chars of string constant, but rely on `n` (size) being accurate; no explicit null-termination checks for `token` text in ICON/FCON cases.
2. **Sentinel value semantics**: `errcnt = -1` to mark "too many errors" is a bit cryptic; modern code might use an enum or boolean flag.
3. **Global token state exposure**: functions like `printtoken()` directly read global `t`, `token`, `tsym`, `cp` without validation—assumes parser maintains invariant that these are in sync.
4. **Format specifier portability**: custom `%k`, `%w`, `%S` codes in `fprint` calls are not standard C; requires a custom wrapper implementation (not in this file, so assume it's in a util library).
