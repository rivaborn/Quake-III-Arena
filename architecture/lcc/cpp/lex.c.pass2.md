# lcc/cpp/lex.c — Enhanced Analysis

## Architectural Role

This file implements the **lexical analysis stage** of the LCC C preprocessor—a critical early step in the offline compilation pipeline that feeds QVM bytecode used by the engine's VM system. It sits upstream of all game logic compilation (cgame, game, ui VMs); the tokenization it produces flows directly into the preprocessor's parser and downstream semantic analysis. No runtime engine code depends on this file directly, but all QVM artifacts indirectly depend on its correctness.

## Key Cross-References

### Incoming (who depends on this file)
- **lcc/cpp/cpp.c** (preprocessor main): calls `expandlex()` at startup to initialize FSM tables, calls `fixlex()` to configure C++ comment mode based on target language
- **lcc/cpp/cpp.c** (preprocessor frame loop): repeatedly calls `gettokens()` to fill token buffers for each source line
- **lcc/cpp/include.c**: likely calls `gettokens()` when processing included files

### Outgoing (what this file depends on)
- **lcc/cpp/cpp.h**: for `Token`, `Tokenrow`, `Source` struct definitions and preprocessor constants (`UNCLASS`, `NAME`, `NUMBER`, etc.)
- **Platform I/O**: `fillbuf(s)` to refill the input buffer (likely in `lcc/cpp/cpp.c`)
- **lcc/cpp/cpp.c**: error reporting via `error()`, trigraph/line-folding helpers (`trigraph()`, `foldline()`), debug buffer (`outbuf[]`)

## Design Patterns & Rationale

**Offline FSM Compilation**: The `fsm[]` spec array is human-readable (each row declares state → chars → nextstate), then `expandlex()` expands it into `bigfsm[256][MAXSTATE]` at startup. This trades one-time initialization cost for O(1) token classification per character during lexing—essential for performance in a compiler tool processing large source files.

**Character Class Abstraction**: `C_ALPH`, `C_NUM`, `C_WS`, `C_XX` encode ranges (e.g., `[a-zA-Z_]`, `[0-9]`) compactly in the spec, then `expandlex()` unpacks them into individual byte entries. This keeps the source table readable while enabling fast lookup.

**Lookahead Encoding**: `S_SELF` vs `S_SELFB` distinguish tokens that consume the current character (`S_SELF`) from those that leave it for the next token (`S_SELFB`). The `QBSBIT` flag (bit 6) marks states requiring special character handling (trigraphs, line-folding) for deferred processing.

**Dual-Buffer State Machine**: The `Source` structure carries `inb`/`inl`/`inp` (buffer base, limit, position) across calls, allowing `gettokens()` to refill mid-line without losing state. On reset, it compacts the buffer if the input pointer drifts far enough (line 238: `if (ip >= s->inb+(3*INS/4))`).

## Data Flow Through This File

1. **Initialization** (`expandlex()`): Expands the compact `fsm[]` spec into the dense 256×32 `bigfsm` lookup table
2. **Per-Line Tokenization** (`gettokens()`):
   - Input: source bytes from `cursource->inp` (current read position in buffer)
   - Token by token: FSM state machine driven by `bigfsm[char][state]`
   - Special cases (trigraphs, line continuations, very long comments) trigger error recovery
   - Output: `Token` stream written to `Tokenrow->lp` array, including `type`, `len`, `wslen`, `t` (string pointer)
3. **Buffer Management**: When buffer exhausted, `fillbuf(s)` refills; `memmove()` compacts if pointer drifts
4. **Return value**: `nmac` flag indicating whether any potential macro names were seen (fast path optimization for preprocessor)

## Learning Notes

**C Preprocessor Maturity**: This lexer handles multiple C standards in one pass—trigraph sequences (`??=` → `#`), C89 block comments, C99/C++ line comments (`//`), and string/char literal escaping. The modular FSM design allows the downstream parser to remain agnostic to these variations.

**No Recursive Descent**: Unlike modern compiler front-ends, there is no recursive descent parser here—the entire preprocessing pipeline is single-pass, table-driven, and streaming. This reflects 1990s compiler tool design priorities: minimal memory footprint, predictable performance on embedded systems.

**Buffer I/O as First-Class**: The `Source` struct embedding `inb/inl/inp` mirrors how Quake engines (including the runtime) manage streams—buffered, position-tracked, with manual lifecycle. This pattern appears throughout the engine (e.g., `FS_Read` in `qcommon/files.c`).

**Idiomatic Tricks**: Line 255's `state&0177` masks off the upper bits (which encode token type); line 339's `ip-1` backtracks for recovery. These bit-twiddling idioms were common in older C codebases to compress state and action into single integers.

## Potential Issues

- **Unchecked Buffer Overflow** (line 303): Very long comments trigger `memmove(tp->t, ip, ...)` without validating that `tp->t` points into the token buffer rather than the source buffer—could corrupt nearby structures.
- **Trigraph/Folding Loop** (line 490): `while (s->inp+2 >= s->inl && fillbuf(s)!=EOF)` will spin infinitely if input is exactly 2 bytes from limit with no more data available.
- **Character Class Bounds** (line 438): Loop `for (j=0; j<=256; j++)` in `C_ALPH` case is off-by-one (should be `j<256`), causing `bigfsm[256]` to be written.
