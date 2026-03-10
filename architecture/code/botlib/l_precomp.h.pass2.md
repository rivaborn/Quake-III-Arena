# code/botlib/l_precomp.h — Enhanced Analysis

## Architectural Role

This header exposes botlib's C-style preprocessor—a foundational utility subsystem that tokenizes and macro-expands bot script files before higher-level parsers consume them. It bridges the gap between raw script source (character definitions, weapon configs, chat templates) and semantic bot AI systems, handling `#define` macros, `#if/#ifdef` conditional compilation, and `#include` file inclusion. The preprocessor is part of botlib's self-contained utility stack (`l_script.c`, `l_libvar.c`, `l_memory.c`) and integrates closely with the VM boundary via a handle-based API, allowing the game DLL to parse bot configs without holding raw engine pointers.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/botlib/be_ai_*.c` (chat, goal, move, weap):** Call `LoadSourceMemory` / `LoadSourceFile` / `FreeSource` to load and parse bot personality scripts, weapon selection tables, and chat response templates
- **`code/botlib/be_interface.c`:** Exposes the handle-based API (`PC_LoadSourceHandle`, `PC_ReadTokenHandle`, `PC_SourceFileAndLine`) to the game VM via `botlib_export_t` vtable; game DLL uses these to load .c files compiled as bot scripts
- **`code/bspc/be_aas_bspc.c`:** Reuses the same `source_t` / `PC_ReadToken` pipeline in the offline BSP→AAS compiler to parse map entity strings and AAS config files

### Outgoing (what this file depends on)
- **`code/botlib/l_script.h`:** Provides `token_t`, `script_t`, `punctuation_t` core token and script abstractions used by `source_t`
- **`code/botlib/l_memory.c`:** Heap allocation for `define_t` nodes, `source_t`, `indent_t` stack frames, and lookahead token buffers
- **`code/botlib/l_log.c`:** Called by `SourceError` / `SourceWarning` to emit formatted diagnostics
- **`q_shared.h`:** Provides `MAX_QPATH`, `QDECL`, and the `pc_token_t` fallback definition for BSPC builds

## Design Patterns & Rationale

**Separation of Concerns:**  
Tokenization (this file) is decoupled from semantic parsing (consumer modules like `be_ai_chat.c`). The preprocessor handles all syntactic preprocessing—macro expansion, conditional directives, token classification—leaving the caller free to implement domain-specific grammar. This mirrors C's own compiler architecture.

**Hash-Chained Define Table:**  
Macro lookup is O(1) via `source->definehash` chains. Since `PC_ReadToken` is called in a hot loop during parsing, this cheap lookup is critical to performance. Global defines (injected via `PC_AddGlobalDefine`) are stored separately and merged at source load time.

**Script Stack + Indent Stack:**  
`#include` nesting is handled by stacking `script_t` pointers in `source->scriptstack`. Conditional compilation depth is tracked in a parallel `indent_t` stack, allowing `PC_ReadToken` to skip tokens when inside a false `#if` block. This design avoids building the full token tree in memory before parsing.

**Handle-Based API for VM Boundary:**  
Functions like `PC_LoadSourceHandle` map opaque `int` handles to a static table of `source_t*` pointers. This protects the engine from buggy VM code holding stale pointers and allows the engine to unload sources on VM shutdown. The handle API is the **only** preprocessor surface the game DLL sees.

**Path Separator Macros:**  
Pre-C99 cross-platform support using `#if defined(WIN32)|defined(_WIN32)...` guards and static `#define` strings. Modern code would use runtime platform detection or platform-specific filesystem APIs, but this approach avoids runtime overhead.

## Data Flow Through This File

**Entry Points (by use case):**
1. **Script file loading** → `LoadSourceFile(filename)` → reads `.c` file from disk, allocates `source_t`, initializes script stack and define hash table
2. **In-memory script loading** → `LoadSourceMemory(ptr, len, name)` → wraps a buffer in a `source_t`; used when scripts are already compiled into the executable (BSPC tool)
3. **Token reading loop** → `PC_ReadToken(source, token)` → expands macros, processes `#if`/`#else`/`#endif`, returns next preprocessed token to caller

**Transformation:**
- Raw script source (text) → tokenized stream (respecting macro definitions and conditional blocks)
- `#define MACRO_NAME` tokens → registered in `source->definehash`; future token stream skips definition body
- `#if CONDITION` / `#endif` blocks → tracked on `source->indentstack`; tokens inside false blocks skip to next `#else`/`#elif`/`#endif`
- `MACRO_NAME(arg1, arg2)` invocations → token substitution with argument binding

**Exit Points:**
- `token_t` stream flows to higher-level parsers in `be_ai_chat.c`, `be_ai_weap.c`, etc., which build syntax trees (e.g., chat response templates)
- Handle-based API returns opaque `int` handles to the game VM; VM calls `PC_ReadTokenHandle` to advance, `PC_SourceFileAndLine` for debug info

## Learning Notes

**Idiomatic to Quake III Era (vs. Modern Practice):**

| Aspect | Q3A (this file) | Modern Practice |
|--------|-----------------|-----------------|
| **Config Format** | C-like `#define` + custom syntax (`.ai`, `.c` files) | JSON, YAML, Lua, or custom AST-based format |
| **Preprocessing** | Full C preprocessor subset in library form | Integrated into single lexer/parser; no separate tokenizer |
| **Macro System** | Hash-chained `#define` table with name + token-list | Configuration objects or named constants; no textual substitution |
| **Conditional Compilation** | `#ifdef` / `#if` stack-based skip logic | Runtime conditionals; no compile-time branching |
| **VM Boundary** | Integer handle indirection for safety | Serialized data structures or RPC; direct pointer access avoided |

**Concepts Illustrated:**
- **Two-phase pipeline:** Preprocessing (expand macros, conditionals) → parsing (semantic structure). Modern engines collapse this into one phase.
- **Symbol table caching:** Hash chains for O(1) define lookup is a foundational technique still used (e.g., in GPU shader compilers).
- **Conditional compilation:** The indent-stack pattern mirrors how C compilers implement `#if`/`#endif` tracking without materializing skipped tokens.

## Potential Issues

1. **Global Define Pollution:**  
   `PC_AddGlobalDefine` affects all subsequently opened sources. If a bot script adds a global define, it persists across multiple game rounds unless explicitly cleared. No automatic cleanup on source free. Could cause subtle cross-script bugs if macro names collide.

2. **Handle Table Saturation:**  
   The handle-based API likely uses a fixed-size handle table (not shown in header). If the game VM opens many sources without closing them (leak), handle IDs could wrap or exhaust. `PC_CheckOpenSourceHandles` detects this post-hoc, but there's no preemptive quota.

3. **Recursive Macro Expansion Not Specified:**  
   The header doesn't document whether macros can invoke other macros or recurse. If `#define A B` and `#define B A`, the expansion behavior is unclear (infinite loop? early termination?). Modern preprocessors explicitly guard against this.

4. **No Pragma / Extension Mechanism:**  
   Unlike real C, there's no `#pragma` or vendor extension escape hatch. If a bot config needs special syntax, it must be hard-coded into `PC_ReadToken` logic, mixing domain-specific and generic preprocessing.
