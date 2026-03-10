# lcc/src/input.c — Enhanced Analysis

## Architectural Role

This file implements the lexical input layer for the LCC compiler, which compiles Quake III game code (cgame, game, ui modules) into QVM bytecode. It manages source file buffering, line-number tracking, and preprocessor directive interception (`#pragma ref`, `# n "file"`), serving as the bridge between the filesystem and the compiler's tokenizer. The input layer must coordinate with the compiler's global symbol table (`tsym`) and code generation to emit accurate debugging information.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler main loop** (`lcc/src/input.c:input_init`, `nextline`, `fillbuf`) — called during source file initialization and per-frame lexical scanning
- **LCC tokenizer** (`lcc/src/lex.c` or similar) — calls `nextline()` after consuming a line of tokens; relies on `line`, `lineno`, `file` globals for error reporting
- **Preprocessor/pragma handling** — `pragma()` calls `gettok()` to tokenize `#pragma ref id...` directives and update symbol reference counts via `tsym->ref++`

### Outgoing (what this file depends on)
- **`main_init(argc, argv)`** — called from `input_init()` to initialize compiler subsystems (likely in `lcc/src/main.c`)
- **`gettok()`** — tokenizes pragma directive arguments; returns token type and sets global `token` buffer
- **Compiler globals** — `t` (token type), `token` (token string), `tsym` (current symbol), `src` (source location), `Aflag` (warning level)
- **Standard I/O** — `fread()`, `feof()`, `exit()` for buffered file reading

## Design Patterns & Rationale

- **Circular double-buffer**: Splits `buffer[MAXLINE+1 + BUFSIZE+1]` into a "holding zone" (`[0...MAXLINE]`, for line-start lookback) and an "active zone" (`[MAXLINE+1...]`, for fresh disk reads). When `cp >= limit`, the unprocessed tail is shifted left and fresh data appended. This avoids re-reading and allows flexible line length without allocating per-line.

- **Preprocessor line directive integration** (`#pragma`, `# lineno "file"`): Rather than parsing in a separate pass, the input layer intercepts these during line scanning. `resynch()` parses `#pragma ref id...` to increment symbol reference counts, and `# n "file"` to update compiler state for error location tracking—crucial for accurate source mapping when debugging QVM code.

- **Goto-based parser** (`line:` label in `resynch()`): The `#line` and `# n` directives share parsing logic; using a label/goto avoids code duplication in 1990s-style C.

## Data Flow Through This File

1. **Initialization**: `input_init()` → `main_init()`, then `fillbuf()` loads first disk block
2. **Per-token**: Lexer calls `nextline()` when advancing to next source line
3. **Buffer refill**: If `cp >= limit`, `fillbuf()` shifts unconsumed tail, reads next BUFSIZE bytes from stdin, updates `limit`
4. **Directive interception**: When `*cp == '#'` (after stripping leading whitespace), `resynch()` is invoked:
   - `#pragma ref id...` → increment `tsym->ref`, call `use(tsym, src)` for reference tracking
   - `# lineno "filename"` → update `lineno`, `file`, `firstfile` (for cross-file line mapping)
5. **Output**: Global `line`, `lineno`, `file` pointers/values feed to error reporting and debug symbol generation

## Learning Notes

- **Compiler input buffering pattern**: This is the canonical approach from the 1980s–90s (LCC was written by Fraser & Hanson circa 1991). Modern lexers often use memory-mapped files or incremental tokenization, but the circular-buffer technique minimizes I/O and memory allocation.

- **Pragma as a compiler hook**: The `#pragma ref` syntax is non-standard; it's a Quake III–specific extension to track symbol references for dead-code elimination or cross-module analysis. This shows how compilers can embed domain-specific metadata in pragmas.

- **Line tracking across preprocessing**: In traditional two-pass compilers, preprocessor runs first and emits line directives. LCC integrates this online, updating `lineno` and `file` on-the-fly. This is simpler but tightly couples the lexer to source mapping.

- **Sentinel-based loop termination**: `*limit = '\n'` acts as a sentinel to avoid bounds checking in tight loops—a micro-optimization from the pre-modern C era.

## Potential Issues

- **Confusing loop condition** (line 30): `while (*cp == '\n' && cp == limit)` appears to conflate two conditions. If `cp == limit`, then `*cp` is the sentinel `'\n'`, so the loop would be infinite. This likely should be `while (*cp == '\n' || cp == limit)` or has a missing negation.

- **No line-length validation**: If a source line exceeds `MAXLINE`, the buffer management in `fillbuf()` could overflow or corrupt state. Modern lexers validate this explicitly.

- **Single-file assumption**: The input system reads only from `stdin`, not arbitrary file handles. The `file` global tracks *which* file a line came from (via `#` directives), but actual multi-file input would require changes to `fillbuf()`.
