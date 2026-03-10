# common/scriplib.h — Enhanced Analysis

## Architectural Role

This header defines the **core text parsing infrastructure for the entire offline build-pipeline toolchain**. It is shared by q3map (BSP compiler), bspc (AAS compiler), and q3radiant (level editor) to parse `.map` files, `.shader` definitions, AAS configs, and other structured text assets. Unlike the runtime engine (which uses binary BSP files), these tools work with source text, making a robust tokenizer essential to the compilation workflow. The design is deliberately simple: single-threaded, stateful, global parse cursor—appropriate for command-line tools that ingest one file at a time without context stacking.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map** (`q3map/*.c`) — parses `.map` files and `.shader` scripts; uses `LoadScriptFile`, `GetToken`, `Parse*Matrix` functions
- **bspc** (`code/bspc/*.c`) — parses AAS configuration files and map source data; reuses the same tokenizer
- **q3radiant** (`q3radiant/*.cpp`) — the level editor; reads/writes `.map` files via this interface
- **Common build utilities** (`common/aselib.c`, `common/bspfile.c`, etc.) — sister utilities that follow the same tokenizer pattern; share the same foundational design philosophy

### Outgoing (what this file depends on)
- **cmdlib.h** — provides `qboolean`, error/file I/O functions (`LoadFile`, `Error`), foundational string/memory utilities
- **mathlib.h** — provides `vec_t` (floating-point scalar, either `float` or `double`), matrix dimension constants, vector utilities
- **(Implicit) stdio.h** — `FILE *` for the Write* functions

## Design Patterns & Rationale

**Stateful Single-Context Parser:** Rather than passing a context object through a call stack, this design uses module-scope globals (`scriptbuffer`, `script_p`, `scriptend_p`, etc.). This pattern is idiomatic for 1990s–2000s C tools and works perfectly when:
- Only one file is parsed at a time (true for map/shader compilation)
- Tools are single-threaded (true for offline tools in this era)
- Simplicity is valued over flexibility

The tradeoff: no support for nested file inclusion or concurrent parses. This is intentional—callers must finish one script, call `LoadScriptFile` again, and all state resets.

**Single-Level Lookahead:** `UnGetToken` provides only one token of pushback. This is sufficient for a simple recursive-descent parser (e.g., `GetToken` → check result → decide on branch → optionally `UnGetToken` to reconsider). Most parsers in the tools use this pattern rather than deeper lookahead.

**I/O Symmetry:** The symmetric `Parse*Matrix` / `Write*Matrix` family reflects the roundtrip requirement: tools must read `.map` source → modify it → write it back byte-for-byte compatible. This is why matrix serialization format is carefully preserved.

**Line Tracking:** `scriptline` is maintained throughout parsing to enable error messages with precise line numbers—critical for a level editor or compiler frontend that reports parse errors to users.

## Data Flow Through This File

1. **Load Phase:** Caller invokes `LoadScriptFile(filename)` or `ParseFromMemory(buffer, size)` → script is buffered in memory, `script_p` points to start, `endofscript = qfalse`.

2. **Parse Phase:** Caller loop:
   - `GetToken(qboolean crossline)` → `token[]` populated, `script_p` advances past whitespace/comments
   - Caller examines `token[]` and decides what to do
   - If wrong token: `MatchToken(expected_string)` aborts with error, or caller calls `UnGetToken()` to retry
   - If optional field: `TokenAvailable()` peeks without consuming

3. **Matrix I/O:** For structured data (e.g., transformation matrices in `.map` entity definitions):
   - **Parse**: `ParseXDMatrix(dims, output_array)` reads parenthesized floats from token stream
   - **Write**: `WriteXDMatrix(FILE*, dims, input_array)` serializes same format back to disk

4. **End of Script:** When `GetToken()` returns `qfalse` or hits `scriptend_p`, `endofscript = qtrue` and parse loop terminates.

## Learning Notes

**Idiomatic to This Engine / Era:**
- **Global mutable state** is the default in this codebase (compare to `cmdlib.h`, `mathlib.h`). Modern engines use context objects and dependency injection, but Q3A's tools reflect late-1990s C conventions.
- **Fixed-size buffers** (`token[MAXTOKEN]`) rather than dynamic strings. This reflects assumptions about maximum token length (1024 chars was safe for Q3 shaders and map files).
- **No error recovery.** `MatchToken` calls `Error()` and aborts the entire tool on parse failure. Modern parsers accumulate errors and continue; this one stops at first error, which is reasonable for a compiler.

**Connections to Engine Programming:**
- **Recursive-descent parsing**: While not explicitly shown here, the header documents a classic recursive-descent paradigm. Tools use `GetToken` to implement grammar productions recursively. This is how `.map` entity definitions and `.shader` multi-pass combos are parsed.
- **Serialization roundtrips**: The I/O symmetry (`Parse*/Write*` pairs) echoes principles of serialization frameworks in modern engines; this is a hand-coded version of those principles.
- **Arena allocation**: The tools reuse a single global `scriptbuffer` per file, which is a form of arena allocation. Once a file is parsed, the entire buffer is freed at `LoadScriptFile` time.

## Potential Issues

1. **No Nested Inclusion Support**: If `.map` files or `.shader` definitions ever need to `#include` other files, this design would require a stack of parse contexts. Currently not supported.

2. **Token Buffer Overflow**: If a single token exceeds `MAXTOKEN` (1024), behavior is undefined. Modern tools use dynamic allocation or reject oversized tokens gracefully.

3. **Stateful Global Conflicts**: If multiple threads or coroutines in a tool tried to use this simultaneously, `script_p` and `token[]` would be corrupted. Single-threaded assumption is implicit and not enforced.

4. **No Character Rewind Beyond One Token**: `UnGetToken` only rewinds one token. If parse logic needs multi-token lookahead, it must re-architect or buffer more state locally.

These are **not bugs** for the intended use case (single-threaded CLI tools). They reflect deliberate scope limitations.
