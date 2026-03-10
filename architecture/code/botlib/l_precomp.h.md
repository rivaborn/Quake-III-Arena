# code/botlib/l_precomp.h

## File Purpose
Declares the public interface for the botlib's C-style preprocessor, which tokenizes and macro-expands script/config files used by the bot AI system. It provides `#define`, `#if`/`#ifdef`/`#ifndef`/`#else`/`#elif` conditional compilation, and `#include` support for bot script parsing.

## Core Responsibilities
- Define data structures for macro definitions (`define_t`), conditional indent tracking (`indent_t`), and source file state (`source_t`)
- Declare token reading and expectation functions used by higher-level bot script parsers
- Declare macro/define management (per-source and global)
- Declare source file loading from disk or memory
- Expose a handle-based API (`PC_LoadSourceHandle` etc.) for use via the engine's trap/syscall interface
- Provide cross-platform path separator macros and BSPC build compatibility shims

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `define_t` | struct | Represents a `#define` macro: name, flags, builtin ID, parameter list, token expansion list, and hash-chain links |
| `indent_t` | struct | Tracks one level of conditional compilation (`#if`/`#ifdef` etc.) on a stack; holds skip flag and owning script |
| `source_t` | struct | Complete state of an open source being parsed: script stack, token lookahead, define table + hash, indent stack, last token |
| `pc_token_t` | typedef/struct | Simplified token struct exposed via the handle-based API (conditionally defined for BSPC when `q_shared.h` is absent) |

## Global / File-Static State

None declared in this header (global define list managed in `l_precomp.c`).

## Key Functions / Methods

### PC_ReadToken
- Signature: `int PC_ReadToken(source_t *source, token_t *token)`
- Purpose: Read the next preprocessed token from a source, expanding macros and processing directives.
- Inputs: Active `source_t`, output `token_t *`
- Outputs/Return: Non-zero on success, 0 on EOF or error
- Side effects: Advances `source->scriptstack`, may push/pop `source->indentstack`
- Calls: Defined in `l_precomp.c`
- Notes: Honors lookahead tokens in `source->tokens`

### PC_ExpectTokenString / PC_ExpectTokenType / PC_ExpectAnyToken
- Signature: `int PC_ExpectTokenString(source_t*, char*)` / `int PC_ExpectTokenType(source_t*, int, int, token_t*)` / `int PC_ExpectAnyToken(source_t*, token_t*)`
- Purpose: Assert the next token matches a specific string or type; emit a source error if not.
- Notes: Convenience wrappers over `PC_ReadToken` used by higher-level parsers for grammar enforcement.

### PC_CheckTokenString / PC_CheckTokenType
- Signature: `int PC_CheckTokenString(source_t*, char*)` / `int PC_CheckTokenType(source_t*, int, int, token_t*)`
- Purpose: Peek/consume next token only if it matches; non-fatal alternative to Expect variants.
- Outputs/Return: Non-zero if matched and consumed, 0 otherwise.

### PC_AddDefine / PC_AddGlobalDefine / PC_RemoveGlobalDefine / PC_RemoveAllGlobalDefines
- Signature: `int PC_AddDefine(source_t*, char*)` / `int PC_AddGlobalDefine(char*)` / `int PC_RemoveGlobalDefine(char*)` / `void PC_RemoveAllGlobalDefines(void)`
- Purpose: Manage per-source and process-wide macro definitions injected before parsing begins.
- Side effects: Allocates/frees `define_t` nodes; modifies a global define list in `l_precomp.c`.

### LoadSourceFile / LoadSourceMemory / FreeSource
- Signature: `source_t *LoadSourceFile(const char*)` / `source_t *LoadSourceMemory(char*, int, char*)` / `void FreeSource(source_t*)`
- Purpose: Allocate and initialize a `source_t` from a file path or in-memory buffer; release all associated resources.
- Side effects: File I/O (LoadSourceFile), heap allocation/deallocation.

### PC_LoadSourceHandle / PC_FreeSourceHandle / PC_ReadTokenHandle / PC_SourceFileAndLine
- Signature: `int PC_LoadSourceHandle(const char*)` / `int PC_FreeSourceHandle(int)` / `int PC_ReadTokenHandle(int, pc_token_t*)` / `int PC_SourceFileAndLine(int, char*, int*)`
- Purpose: Integer-handle API wrapping the pointer-based API, suitable for use across the VM/engine syscall boundary.
- Notes: `PC_CheckOpenSourceHandles` detects and reports leaked handles.

### SourceError / SourceWarning
- Signature: `void QDECL SourceError(source_t*, char*, ...)` / `void QDECL SourceWarning(source_t*, char*, ...)`
- Purpose: Emit formatted error/warning messages with file name and line number context from the source.
- Side effects: Output to engine console/log.

## Control Flow Notes
This header is consumed at init time by bot AI subsystems that load script files (character scripts, weapon configs, chat scripts). `LoadSourceFile` / `LoadSourceMemory` are called during bot initialization; `PC_ReadToken` and Expect/Check helpers are called in a parse loop; `FreeSource` is called at shutdown or when a script is fully consumed. The handle-based API is called from `be_interface.c` to service trap calls from the game VM.

## External Dependencies
- `token_t`, `script_t`, `punctuation_t` — defined in `l_script.h`
- `MAX_QPATH` — defined in `q_shared.h`
- `QDECL` — defined in `q_shared.h` (or stubbed for BSPC)
- `pc_token_t` — defined in `q_shared.h` when not building BSPC, or locally here under the BSPC guard
