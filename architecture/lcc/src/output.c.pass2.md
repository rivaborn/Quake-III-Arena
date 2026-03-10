# lcc/src/output.c — Enhanced Analysis

## Architectural Role
This file implements the formatted output subsystem for the LCC C compiler. It provides the compiler infrastructure with printf-style functions supporting custom format specifiers tailored to the compiler's domain (tokens, types, source coordinates). As part of the **offline LCC compiler toolchain**, this file has no runtime engine dependencies—it exclusively serves the compilation pipeline that transforms C source into QVM bytecode.

## Key Cross-References

### Incoming (who depends on this file)
- Other `lcc/src/` compiler modules (e.g., `lcc/src/error.c`, `lcc/src/main.c`, diagnostic generators) call `print()`, `fprint()`, `stringf()` for console output and debug reporting
- No calls from runtime engine code (`code/*/`, `common/`, `q3map/`, `q3radiant/`, etc.)

### Outgoing (what this file depends on)
- `stdio.h` (FILE, fputs, fputc, sprintf, putc) — C standard library only
- `string.h` implicitly (via token.h inclusion in `vfprint`)
- Compiler-internal token definitions (`token.h`) for `%k` specifier
- Compiler-internal type system (`outtype()` function, assumed from `c.h`)
- No dependencies on qcommon, renderer, or any runtime engine modules

## Design Patterns & Rationale

**Dual-mode output buffer (FILE\* or char\*)**: Each output function (`outs`, `outd`, `outu`) accepts both a `FILE *f` and a `char *bp` (buffer pointer). If `f` is non-NULL, output goes to the file; otherwise, characters accumulate in the buffer. This single-code-path design eliminates duplication: `print()` calls `vfprint(stdout, NULL, ...)` for console output, while `stringf()` calls `vfprint(NULL, buf, ...)` to build a string. Elegant for a compiler that must both log diagnostics and assemble error messages.

**Custom format specifiers**: Beyond standard printf (`%d`, `%s`, `%x`), the compiler defines domain-specific formats:
- `%k` — token name lookup (compiler-specific vocabulary)
- `%t` — type pretty-printing (calls `outtype()`)
- `%w` — source location (file:line format for error messages)
- `%S` — bounded substring (length-prefixed string)
- `%I` — indentation (repeated spaces, likely for AST/IR pretty-printing)

This avoids string manipulation in call sites and keeps formatting logic centralized.

**Backward-building integer conversion** (in `outd`/`outu`): Strings are built **backwards** into a local buffer, starting at the end and working toward the beginning. The final position is then output. This avoids computing string length upfront and is a classic manual itoa implementation.

## Data Flow Through This File

1. **Entry points**: Compiler calls `print()`, `fprint()`, or `stringf()` with format string and variadic args
2. **Format parsing**: `vfprint()` scans the format string character-by-character, dispatching on `%` specifiers
3. **Type-specific output**: For each specifier, extract the appropriate type from `va_list` and call the matching output helper (`outd`, `outu`, `outs`, etc.)
4. **Unified accumulation**: All helpers write to the same buffer pointer or FILE, maintaining position across specifiers
5. **Termination**: If building a buffer (not FILE), null-terminate and call `string()` to intern it (in `stringf`)

Example flow for `stringf("variable %s at %w", name, coord)`:
- Calls `vfprint(NULL, buf, ...)`
- Parses `variable ` → writes to buf, advances bp
- Parses `%s` → extracts `char*` from va_list, calls `outs()` → accumulates into buf
- Parses `%w` → extracts `Coordinate*`, formats file:line, accumulates into buf
- Null-terminates, returns `string(buf)` (a compiler-internal string pool intern)

## Learning Notes

**For compiler writers**: This demonstrates the standard pattern for implementing printf-style functions with custom specifiers. The dual-mode (FILE vs. buffer) approach is economical when a tool must support both immediate output and deferred string construction.

**Era-specific idioms**: The backward-building integer conversion and manual va_list handling reflect pre-1990s compiler infrastructure. Modern implementations would use `snprintf()` or format libraries, but LCC was written in a time when portable variadic handling and format libraries were less standardized.

**Separation of concerns**: The file maintains a clean division: `outs/outd/outu` handle **mechanics** (where to write, how to format), while `print/fprint/stringf/vfprint` handle **policy** (which entry point, how to initialize buffers). This makes the code resilient to changing platforms or output targets.

**No connection to runtime engine**: Unlike the renderer (`tr_main.c`), server (`sv_main.c`), or client (`cl_main.c`), this compiler utility is **100% decoupled** from the Q3 engine. It could be used to compile unrelated languages to different VM bytecodes without engine modifications.

## Potential Issues

- **Buffer overflow in `stringf`**: The hardcoded 1024-byte `buf` array (line 58) will silently truncate or corrupt if a formatted string exceeds that size. No bounds checking in `vfprint()`. A sufficiently complex error message (e.g., deeply nested type with long symbol names) could overflow.
- **`sprintf` in floating-point case (line 84)**: Uses dynamically-constructed format string (`format[1] = *fmt`), which works but is fragile if extended (e.g., adding `%L` for long double would break the single-character assumption).
