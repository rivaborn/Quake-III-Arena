# lcc/cpp/cpp.h — Enhanced Analysis

## Architectural Role

This file defines the core data structures and public interface for the LCC C preprocessor, a standalone component of the offline build toolchain (not runtime engine). The preprocessor tokenizes and expands C source files before compilation to QVM bytecode. It's used exclusively during the build phase to process `code/game/`, `code/cgame/`, and `code/ui/` source files before they're compiled to VM bytecode by the LCC compiler; the runtime engine never directly invokes it.

## Key Cross-References

### Incoming (who depends on this file)
- **cpp.c, eval.c, lex.c, macro.c, etc.** in `lcc/cpp/` implement the functions declared here
- **code/bspc/be_aas_bspc.c** and other BSPC tool code may indirectly use preprocessor services during offline AAS compilation
- Build system scripts reference LCC's cpp as part of the VM compilation pipeline (game.q3asm, cgame.q3asm, ui.q3asm)

### Outgoing (what this file depends on)
- **Platform layer** (`code/unix/`, `code/win32/`): `open()`, `close()`, `read()`, `write()`, `dup2()`, `creat()` — file I/O primitives declared at end of header
- **Standard C library**: `size_t`, memory management conventions
- **No engine dependencies**: This is a build tool, not linked into runtime executables

## Design Patterns & Rationale

**Token Stream Processor**: Core data flow is `Source → Tokenrow → process() → puttokens()`. Tokens carry metadata (type, hideset, whitespace length) enabling intelligent macro expansion without token fusion errors.

**Hideset Mechanism** (via `hideset` bitfield in `Token`): Prevents infinite recursion when a macro expands to itself. Standard preprocessor technique; the `namebit[077+1]` lookup table and `quicklook()`/`quickset()` macros provide O(1) hideset membership testing using 64-bit bitmask per character pair.

**Symbol Table (`Nlist`)**: Separates macro definitions (`vp` — token value, `ap` — argument list) from preprocessor-only names (`val`, `flag`). Enables both object-like and function-like macro support plus built-in directives.

**Double-Buffered Input** (`Source.inb`, `inp`, `inl`): Ring buffer of size `INS` (32KB) allows incremental tokenization of arbitrarily large files; `fillbuf()` refills asynchronously.

**Sentinel Bytes** (`EOB=0xFE`, `EOFC=0xFD`): Tokenizers check for these instead of buffer bounds, reducing branch pressure during hot lexing loops.

Why designed this way: The preprocessor must handle:
- Arbitrary nesting of `#include`, `#if`, `#define`
- Macro expansion without token gluing errors (solved by hideset)
- Efficient multi-pass traversal (tokenize once, then reprocess for expansion)
- Minimal memory footprint for embedded systems and old hardware (era-appropriate ~1990s constraints)

## Data Flow Through This File

1. **Input**: File descriptor (`Source.fd`) opened by `setsource()`; raw bytes flow into `Source.inb` via `fillbuf(Source *)`
2. **Tokenization**: `gettokens(Tokenrow *, int)` scans buffer, produces sequence of `Token` structs in allocated `Tokenrow.bp..lp` array
3. **Preprocessing**: `process(Tokenrow *)` dispatches control directives (`#if`, `#define`, `#include`, etc.) via `control()` and macro expansion via `expand()`
4. **Macro Expansion**: 
   - `lookup()` finds `Nlist` entry by name
   - `gatherargs()` collects actual arguments if function-like
   - `substargs()` replaces formal args with actuals in token stream
   - Hideset is updated to prevent re-expansion
5. **Output**: `puttokens(Tokenrow *)` writes expanded token stream to output buffer (`outp`); final flush via `flushout()`
6. **Final**: Compiled preprocessed `.i` file fed to next pipeline stage (C parser/compiler)

State machines:
- **Source stack** (`source→next`): `#include` nesting creates linked list; `unsetsource()` pops stack
- **If-nesting** (`ifdepth`, `ifsatisfied[NIF]`): Tracks conditional compilation state; max 32 levels

## Learning Notes

**What this teaches about preprocessor design:**
- Token-based expansion is more robust than text-based (easier to reason about whitespace, operator precedence)
- Hideset as a bloom filter is elegant for O(1) recursion detection
- Sentinel values (`EOB`, `EOFC`) avoid boundary-condition branches — micro-optimization visible even in 1990s code
- Separating tokenization from expansion allows multi-pass processing and easier debugging

**Idiomatic to this era / different from modern engines:**
- No Unicode awareness (assumes 8-bit ASCII tokens)
- Manual memory management with `domalloc()` / `dofree()` — no garbage collection
- Single-threaded by design; modern preprocessors often parallelize includes
- No optimization for incremental compilation (whole-file reprocessing each build)
- No caching layer for expensive operations like `eval()` of constant expressions

**Conceptual connections:**
- Shares structure with shader preprocessors (Q3 uses `.shader` text files with `#include`, `#ifdef` directives), but that's hardcoded in `code/renderer/tr_shader.c`
- Similar to GLSL/HLSL preprocessors, but for a different domain (VM source, not GPU shaders)
- Not an ECS or scene-graph pattern; pure procedural text transformation

## Potential Issues

**Implicit limitations:**
- Max `NIF=32` nesting levels for `#if` directives — deeply nested headers could overflow
- `NARG=32` argument limit on function-like macros
- No built-in `__VA_ARGS__` support (C99 feature; this assumes C89)
- Hideset uses `int` (16-bit on some platforms); if >16 macros used simultaneously, collisions possible (unlikely in practice, but fragile)

**Not an issue but worth noting:**
- The `extern` function declarations at the end duplicate POSIX/libc signatures without a corresponding `#include` guard — relies on platform-specific build flags to suppress redeclaration warnings

---

This is a **build-time artifact**, not game engine code. Its quality directly affects VM bytecode quality, but the runtime engine never executes or inspects it.
