# common/scriplib.c — Enhanced Analysis

## Architectural Role

This file is a **foundational offline-tool utility** providing tokenization and text parsing infrastructure for all Q3A tool executables (`q3map`, `bspc`, `q3radiant`). It occupies the same tier as `common/aselib.c` (model loading) and `common/bspfile.c` (BSP serialization)—pure tool-time support with zero presence in runtime engine code. The tokenizer forms the backbone of configuration file parsing, shader scripts, and map entity string processing during offline compilation.

## Key Cross-References

### Incoming (who depends on this file)
- **`q3map/` and `bspc/`**: Both tools load BSP maps and configuration files via `LoadScriptFile` → `GetToken` pipeline
- **`q3radiant/`**: Level editor uses script parsing for brush definitions, entity properties, and spline data
- **No cgame/game/client/server calls**: Runtime engine has zero dependencies on scriplib; configuration is baked offline
- **`common/` sibling files**: `aselib.c`, `bspfile.c` follow similar "offline utility" patterns

### Outgoing (what this file depends on)
- **`cmdlib.h`**: `Error()` (fatal, no recovery), `LoadFile()` (heap alloc), `ExpandPath()` (path canonicalization)
- **`scriplib.h`**: Exports all public symbols; declares `vec_t` type pulled from `mathlib.h`
- **Standard C**: `stdio.h` (FILE ops), `stdlib.h` (free), `string.h` (strcmp, strcpy)—no qcommon dependencies
- **No external tool dependencies**: This is self-contained; tools link it directly

## Design Patterns & Rationale

**Stack-based nested parsing context:**
The fixed-size `scriptstack[MAX_INCLUDES=8]` and `script` pointer form a classic single-stack parser implementation. This enables `$include` directive handling without recursive function calls (avoiding deep stack growth during tool startup). The design is intentionally memory-bounded—a safety choice for offline tools where fixed limits are acceptable.

**Global token buffer + look-ahead flag:**
The global `token[MAXTOKEN]` and `tokenready` boolean implement a minimal one-token pushback mechanism. This trades global state for simplicity; modern lexers would use a token queue, but this was pragmatic for early-2000s C.

**Manual line tracking across file boundaries:**
The dual `scriptline` (global) and `script->line` (per-context) counters handle line number propagation correctly when popping includes. Error messages pinpoint source locations accurately—critical for tool debugging.

**Recursive descent matrix parsing:**
`Parse1DMatrix` → `Parse2DMatrix` → `Parse3DMatrix` mirrors the nested-parens structure of data. No intermediate AST; values are written directly to flat arrays. This is lightweight and matches offline tool constraints.

## Data Flow Through This File

```
Disk file or memory buffer
    ↓
LoadScriptFile() or ParseFromMemory()
    ↓ (on GetToken)
Whitespace/comment skipping (recursively advances script_p)
    ↓
Token extraction (quoted or delimited by space/semicolon)
    ↓
$include directive handling (recursive AddScriptToStack)
    ↓
token[] buffer filled; returned to caller
    ↓ (for structured data)
Parse*DMatrix() → atof(token) → flat array storage
```

**Key state transitions:**
- `script_p` advances monotonically within each file's buffer
- Line numbers increment on `\n`; pushed-back files restore `script->line`
- `endofscript` gate prevents token reads after final file closure
- Include files are popped and freed automatically; caller never manages context stack

## Learning Notes

**What a developer studying this learns:**

1. **Offline tool patterns**: Tools have different constraints than engines—fixed memory limits, no frame loop, fatal errors are acceptable. This file exploits those freedoms (global state, no error recovery).

2. **Lexer implementation**: Shows a hand-written tokenizer for a simple grammar (whitespace, comments, quoted strings, unquoted tokens, parenthesis-delimited structures). Real projects often reach for generated parsers; this manual implementation is readable and sufficient for Q3A's config needs.

3. **Include directive handling**: The recursive `$include` + stack approach is the idiomatic way to support nested file inclusion without threading callbacks through the parser. Contrast with modern approaches using include graphs and deduplication.

4. **C era conventions**: No dynamic memory allocation for token buffers (use stack arrays + MAXTOKEN guards), global state (simpler than stateful objects), and `Error()` for fatal conditions (no exceptions in C89).

**What's idiomatic to Quake III, not modern engines:**

- **No lexer/parser separation**: Token extraction and syntactic validation are fused (e.g., `MatchToken` calls `GetToken` inline).
- **Bare pointer arithmetic**: `script->script_p++` and buffer-end checks dominate; no iterator abstractions.
- **Fixed-size limits**: `MAX_INCLUDES=8`, `MAXTOKEN=1024` are compile-time constants; tools fail if exceeded rather than dynamically expanding.
- **Synchronous I/O with `LoadFile`**: No async streaming; entire files are loaded into memory before parsing.

**Connection to game engine concepts:**

While not used by the runtime, this tokenizer shows the **tool-time critical path**: map compilers must parse entity strings, shader definitions, and bot behavior trees—all text formats that benefit from a shared lexer. Modern engines typically decouple tool and runtime (e.g., separate parser libraries), whereas Quake III reused `scriplib` across all tools.

## Potential Issues

1. **Buffer overflow in `strcpy` (line 56, 96)**: Uses unsafe `strcpy(script->filename, ...)`. If `ExpandPath()` returns a string longer than 1023 bytes, stack corruption occurs. Not in the cross-reference context provided, but worth auditing `ExpandPath`.

2. **Comment termination bug (line 201)**: 
   ```c
   while (script->script_p[0] != '*' && script->script_p[1] != '/')
   ```
   Should be `||` (OR) to detect `*/` ending, not AND. Current code requires both `*` and `/` to be *absent* simultaneously—an infinite loop risk if a file ends with `/*` unclosed.

3. **Token size validation**: `MAXTOKEN=1024` is checked at lines 258 and 273, but if `atof()` is called on a token, integer overflow or precision loss could occur silently. Callers assume tokens are valid; no range checking.

4. **Memory buffer mode does not own its buffer**: `ParseFromMemory()` sets `script->buffer` to a caller-provided pointer but does not track ownership. If `EndOfScript()` frees it, corruption results. Current code avoids this (line 147 checks filename != "memory buffer"), but the design is fragile.

5. **Single token lookahead limit**: `UnGetToken()` + second `GetToken()` is the only supported pushback. Parsers needing arbitrary lookahead must restructure their call pattern or build their own token queue.

---

**Sources & Inference:**
- Cross-reference context shows no callsites from `code/game`, `code/cgame`, `code/client`, or `code/server`, confirming tool-only scope.
- Architecture overview places `common/` outside the runtime engine; `scriplib` fits that tier exactly.
- The `vec_t` matrix functions hint at use in geometry tools (`q3map` brush parsing, `q3radiant` spline data).
