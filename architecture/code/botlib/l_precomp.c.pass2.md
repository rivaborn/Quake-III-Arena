# code/botlib/l_precomp.c — Enhanced Analysis

## Architectural Role

This file implements a standalone **C-like preprocessor** that bridges botlib's configuration/script file ecosystem to the game engine. It sits in botlib's internal utility stack (`l_*` layer) and is invoked at **load time** when the server loads bot character files, weapon/item configs, and chat scripts—never during the per-frame game loop. It directly supports **conditional compilation** and **macro expansion** in bot configuration files, allowing level/mod designers to use directives like `#ifdef TEAM_ARENA` or `#define BOT_SKILL` in `.c`-like bot AI config files before those files are consumed by the AI behavioral parser (in `be_ai_char.c`, `be_ai_goal.c`, etc.).

## Key Cross-References

### Incoming (Callers / Dependents)
- **`code/botlib/be_interface.c`** — Primary consumer via public handle-based API (`PC_LoadSourceHandle`, `PC_ReadTokenHandle`, `PC_FreeSourceHandle`, `PC_SourceFileAndLine`). Exposed through `botlib_export_t` vtable to the server.
- **`code/game/g_bot.c`** (via server) — Indirectly: server loads bot configs through botlib, which calls this preprocessor.
- **`code/bspc/be_aas_bspc.c`** — The BSP→AAS compiler reuses this preprocessor when processing bot navigation and map configuration scripts during offline compilation (BSPC build context).

### Outgoing (Dependencies)
- **`code/botlib/l_script.h/c`** — Provides `script_t`, `token_t`, `PS_ReadToken`, `LoadScriptFile`, `LoadScriptMemory`, `FreeScript`, `EndOfScript` (underlying lexer). This file extends the token stream with preprocessing directives.
- **`code/botlib/l_memory.h/c`** — All heap allocation for `define_t` nodes, token copies, indent stack nodes, `source_t` structs.
- **`code/botlib/l_log.h`** — Optional logging in hash-table debug output.
- **`code/qcommon/q_shared.h`** — `Com_Memcpy`, `Com_Memset`, `Com_Error`, `Q_stricmp` (via conditional includes).
- **`code/botlib/be_interface.h`** — `botimport.Print` for error/warning output in BOTLIB context.

## Design Patterns & Rationale

**1. Conditional Compilation / Build Variants**  
The file uses preprocessor conditionals (`#ifdef BOTLIB`, `#ifdef BSPC`, `#ifdef MEQCC`) to create multiple deployments from one source: runtime botlib, offline BSP compiler, and a MEQCC variant. This is idiomatic to Quake era multi-tool builds where shared logic is compiled into different binaries with different headers and error-reporting backends.

**2. Handle-Based Public API**  
External consumers (`be_interface.c`) use integer handles (1–63) instead of direct `source_t*` pointers. The `sourceFiles[64]` static table provides indirection and lifetime management, isolating the preprocessor's internal state from callers. This was a common pattern in the 1990s to provide stable binary interfaces across DLL/static-link boundaries.

**3. Global Define Injection**  
`globaldefines` (a linked list) is cloned into every newly opened source. This allows the server to inject platform/mod-wide definitions (e.g., `#define TEAM_ARENA 1`) that apply to all bot configs without modifying individual files.

**4. Directive Dispatch via Function Pointers**  
Directives like `#define`, `#ifdef`, `#include` are dispatched through the `directives[]` and `dollardirectives[]` arrays, mapping string names to handler functions. This is a lightweight interpreter pattern used throughout this era of engines.

**5. Token-Streaming (Not Tree-Building)**  
Rather than building an AST, the preprocessor outputs a stream of tokens to its caller. Macro expansion pushes expanded tokens back into the source's token queue (`source->tokens`). This is memory-efficient for streaming configuration parsers but constrains what transformations are possible.

## Data Flow Through This File

**Initialization:**
1. Server/tool calls `PC_LoadSourceHandle()` with filename or memory buffer + name.
2. File creates a `source_t`, allocates define hash table, clones all `globaldefines` into it.
3. Pushes initial `script_t` onto the `source->scriptstack`.

**Per-Token Read Loop:**
1. Caller repeatedly invokes `PC_ReadTokenHandle()` (wraps `PC_ReadToken`).
2. `PC_ReadToken` reads from the script via `PC_ReadSourceToken` → `PS_ReadToken`.
3. **Directive Interception:** If token is `#` or `$`, dispatch to `PC_ReadDirective` or `PC_ReadDollarDirective`.
   - `#define NAME ...` → calls `PC_Directive_define`, stores in hash table.
   - `#ifdef NAME` → calls `PC_Directive_ifdef`, pushes indent (skip state) if condition false.
   - `#include "file"` → calls `PC_Directive_include`, pushes new script onto stack.
4. **Macro Expansion:** If token matches a known macro, calls `PC_ExpandDefine` → builds expanded token chain → pushes chain back into `source->tokens` queue for next reads.
5. **Skip State:** Maintains `source->indentstack` to track conditional blocks. If in a skipped region, tokens are consumed but not returned to caller.
6. **String Concatenation:** Adjacent string literals are merged (e.g., `"a" "b"` → `"ab"`).

**Teardown:**
- When source reaches EOF, caller invokes `PC_FreeSourceHandle`.
- `FreeSource` walks `source->scriptstack`, frees all scripts/tokens/indents/defines.

## Learning Notes

**Idiomatic Patterns of This Era:**
- **No Tree/AST:** Early Q3 tooling often operates on token streams, not abstract syntax trees. This is memory-efficient but inflexible.
- **Conditional Compilation as Language Feature:** Using `#ifdef` at the *configuration file* layer (not just the C code layer) is unusual by modern standards; today, we'd use JSON/YAML with application-level conditional logic.
- **Handle Tables for DLL Stability:** The `sourceFiles[]` indirection is a runtime stability workaround before proper opaque-pointer/vtable ABIs were standard.
- **Macro Expansion in Tooling:** The `##` token-merging and `#` stringizing operators are borrowed from C's preprocessor but used in *game data files*, suggesting bot AI configs were originally written in a C-like syntax.

**Connections to Game Engine Concepts:**
- This is a **script pipeline** component: raw text → tokens → semantics (game logic parses tokens into bot configs).
- The **indent stack** (tracking `#if`/`#endif` nesting) is a classic interpreter pattern for conditional execution.
- **Token queues** and **pushback** are fundamental to any streaming parser; the `source->tokens` linked list is a simple but effective way to implement lookahead and macro expansion side effects.

## Potential Issues

1. **Buffer Overflows in Token Merging**  
   `PC_StringizeTokens` and `PC_MergeTokens` use `strcat`/`strncat` without rigorous bounds checking. If a macro expansion produces a very long merged token, it could overflow the fixed `MAX_TOKEN` (likely ~1024 bytes). This is mitigated only by platform-specific `strncat` bounds.

2. **Recursive Include Not Detected Until Runtime**  
   `PC_PushScript` checks for recursive includes by walking the script stack, which is correct but O(n) per include. A set-based check would be more efficient, though this is unlikely to be a bottleneck.

3. **Nested Macro Expansion May Re-expand**  
   If macro body contains a macro name, it gets expanded again during `PC_ExpandDefine`. This is correct C-preprocessor behavior but can lead to infinite loops if someone defines a macro that references itself (e.g., `#define SELF SELF`). No explicit guard is visible in the code.

4. **Global State Fragility**  
   `globaldefines` is a single global pointer shared across all active sources. If two sources are open simultaneously and one modifies this list during iteration (unlikely but possible in multithreaded scenarios), corruption could occur.
