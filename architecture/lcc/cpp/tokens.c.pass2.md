# lcc/cpp/tokens.c — Enhanced Analysis

## Architectural Role

This file is the **token stream abstraction layer** of the LCC C preprocessor, managing tokenized input throughout macro expansion and directive processing. As part of `lcc/cpp/`, it sits at the core of the preprocessing pipeline that converts raw C source (destined for QVM bytecode compilation) into a sequence of preprocessed tokens—before parsing and code generation. The `wstab` optimization reveals this layer's critical role in **correctness during macro substitution**, where whitespace elision can cause harmful token merging.

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/cpp/cpp.c`** (main preprocessor loop) — calls `maketokenrow`, `growtokenrow`, `comparetokens`, `insertrow`, `normtokenrow` during macro expansion and directive handling
- **`lcc/cpp/macro.c`** (macro expansion engine) — consumes token row APIs (`copytokenrow`, `insertrow`, `movetokenrow`, `adjustrow`) when splicing replacement text
- **`lcc/cpp/lex.c`** (lexical analyzer) — feeds tokens into rows via `maketokenrow`
- **`lcc/cpp/eval.c`** (expression evaluator for `#if`) — relies on `puttokens` for debug output
- **All QVM-compilable code** in `code/cgame/`, `code/game/`, `code/ui/` — indirectly, because all preprocessed source passes through this layer

### Outgoing (what this file depends on)
- **`lcc/src/alloc.c`** via `domalloc()` (memory allocation stub) — all token row growth
- **`lcc/cpp/cpp.h`** (header) — `Token`, `Tokenrow`, `wstab[]`, constants (`OBS`, `NL`, `NAME`, etc.)
- **Standard C library** — `memmove`, `memcpy`, `strncmp`, `fprintf`, `write`
- **`lcc/cpp/l_*.c` family** (likely logger/debug utilities) — `flushout()` coordination

## Design Patterns & Rationale

### 1. **Dynamic Array with Exponential Growth** (`growtokenrow`)
```c
trp->max = 3*trp->max/2 + 1;  // 1.5× growth
```
Avoids O(n²) reallocation cost during large macro expansions. Standard amortized O(1) append pattern.

### 2. **Whitespace Tracking as Metadata** (not just stripped)
Each token carries:
- `wslen` — leading whitespace byte count
- `flag & XPWS` — whether whitespace was synthesized during macro expansion

The `wstab[]` table is **the crown jewel**:
- `wstab[type] == 1` for tokens that **never need** preceding whitespace (NL, SBRA `[`, SKET `]`, LP `(`, RP `)`, COMMA, SEMIC, CBRA `{`, CKET `}`)
- Prevents pathological output like `int main( )` → `intmain()` when token sequences merge

**Rationale:** In C preprocessing, **token concatenation happens implicitly**—adjacent tokens in output must remain distinguishable. Macros that substitute into `foo##bar` or simple concatenation must preserve syntactic integrity. Rather than insert spaces unconditionally (bloating output), this table encodes which token *types* are "bracket-like" and thus self-delimiting.

### 3. **Canonical Whitespace Normalization** (`makespace`)
```c
if (tp->flag & XPWS && (wstab[tp->type] || prev_is_safe))
    tp->wslen = 0;  // elide synthesized space if adjacent tokens don't merge
```
Minimizes preprocessor output size while maintaining correctness. Synthesized spaces are stripped if the surrounding tokens are inherently safe.

### 4. **Dual-Pointer Cursor Model**
- `trp->bp` — base (start of token array)
- `trp->tp` — traverse/working pointer (current position in iteration)
- `trp->lp` — limit/end pointer (one past last token)

Allows multiple simultaneous scans and in-place manipulation of token sequences without copying.

## Data Flow Through This File

1. **Input (from lexer):** Raw tokens fed into `Tokenrow` via `maketokenrow()`
2. **Macro Expansion:** `insertrow()` splices replacement text mid-stream
   - `adjustrow()` shifts existing tokens rightward to make space
   - `movetokenrow()` bulk-copies replacement tokens
   - `makespace()` ensures no unwanted merges
3. **Normalization:** `normtokenrow()` creates a clean copy with canonical spacing for caching
4. **Comparison:** `comparetokens()` checks if two token sequences are identical (ignoring `wslen` exact values)
5. **Output (to parser):** `puttokens()` flushes buffered tokens via write(1, ...) to stdout, using a 2× `OBS` size sliding window to batch small writes
6. **Debug:** `peektokens()` prints token stream state to stderr for diagnostics

## Learning Notes

### Idiomatic to 1990s C Compiler Design
- **Tokenization as preprocessing layer:** Pre-tokenizing avoids re-lexing during macro expansion; modular separation of concerns
- **Manual memory management:** `domalloc`/`realloc` reflect era before garbage collection in compiler infrastructure
- **Bit-packing:** `wstab[]` as `char` (not bool/int) saves 56+ bytes—relevant when processing millions of tokens
- **Buffered I/O:** `wbuf` and `write(1, ...)` bypass stdio overhead; critical for preprocessor output throughput

### Modern Comparison
- **ECS/Scene Graphs:** Not applicable; this is a token stream, not a game entity hierarchy
- **Modern preprocessors** (clang, MSVC) use similar token abstraction but with richer metadata (source location, pragmas, token flags)
- **Macro expansion** today often uses recursive descent parser + AST, whereas LCC's approach is stream-based (simpler, suitable for lightweight QVM toolchain)

### Critical Insight: **Correctness Under Macro Expansion**
The `wstab` design solves a subtle problem:
```c
#define CONCAT(a, b) a##b
CONCAT(int, main)  // must produce "intmain", not separate tokens

#define SPACE_SAFE int
SPACE_SAFE main()  // OK, tokens naturally separate
```
Without `wstab`, *every* token substitution would require a space, bloating output and potentially breaking identifiers. With `wstab`, the preprocessor is **minimal yet correct**.

## Potential Issues

1. **No bounds checking in `outnum()`** (line ~350)
   - Assumes buffer `p` is large enough; caller must ensure
   - Not a practical risk if caller allocates `OBS` bytes, but fragile

2. **`write(1, ...)` errors silently ignored** in `puttokens()`
   - If stdout fills or closes, errors are discarded; compilation may complete with truncated output
   - Acceptable for a preprocessor (would fail at next tool in pipeline), but not robust

3. **Whitespace synthesis via `newstring()` allocates separately**
   - Each synthesized space allocates a new string; could fragment memory for token-heavy preprocessing
   - Acceptable given compilation is one-shot and short-lived

4. **`makespace()` modifies token in-place** mid-iteration
   - If an external iterator holds `trp->tp`, concurrent modification could corrupt state
   - Safe in practice because the preprocessor is single-threaded, but unintuitive API

**None are likely to manifest in practice** given QVM source size and single-threaded preprocessor guarantees.
