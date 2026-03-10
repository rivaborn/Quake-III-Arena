# code/botlib/l_precomp.c

## File Purpose
Implements a C-like preprocessor (precompiler) used by the botlib to parse configuration and script files. It handles `#define`, `#include`, `#ifdef`/`#ifndef`/`#if`/`#elif`/`#else`/`#endif`, macro expansion, and expression evaluation for conditional compilation directives.

## Core Responsibilities
- Load and manage a stack of script files (`source_t`), supporting `#include`
- Parse and store macro definitions (`define_t`) with optional parameters, using a hash table for fast lookup
- Expand macros (including stringizing `#` and token-merging `##` operators) into the token stream
- Evaluate constant integer/float expressions in `#if`/`#elif` directives
- Manage conditional compilation skip state via an indent stack
- Expose a handle-based API (`PC_LoadSourceHandle`, `PC_ReadTokenHandle`, etc.) for external consumers
- Maintain a global define list injected into every opened source

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `directive_t` | struct | Associates a directive name string with its handler function pointer |
| `operator_t` | struct | Doubly-linked node for an operator during expression evaluation |
| `value_t` | struct | Doubly-linked node for a numeric value during expression evaluation |

*(Core types `define_t`, `indent_t`, `source_t`, `token_t`, `script_t` are defined in `l_precomp.h` / `l_script.h`.)*

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numtokens` | `int` | global | Debug counter tracking live allocated token count |
| `globaldefines` | `define_t *` | global | Linked list of defines added to every opened source |
| `directives[20]` | `directive_t[]` | file-static | Table mapping `#`-directive names to handler functions |
| `dollardirectives[20]` | `directive_t[]` | file-static | Table mapping `$`-directive names to handler functions |
| `sourceFiles[64]` | `source_t *[]` | file-static | Handle table for externally opened sources (index 1–63) |

## Key Functions / Methods

### LoadSourceFile / LoadSourceMemory
- **Signature:** `source_t *LoadSourceFile(const char *filename)` / `source_t *LoadSourceMemory(char *ptr, int length, char *name)`
- **Purpose:** Allocate and initialize a `source_t`, push the initial script, inject global defines, set up the define hash table.
- **Inputs:** File path or in-memory buffer + length + name.
- **Outputs/Return:** Pointer to new `source_t`, or `NULL` on failure.
- **Side effects:** Allocates heap memory; copies global defines into source.
- **Calls:** `LoadScriptFile`/`LoadScriptMemory`, `GetMemory`, `GetClearedMemory`, `PC_AddGlobalDefinesToSource`

### FreeSource
- **Signature:** `void FreeSource(source_t *source)`
- **Purpose:** Frees all scripts, tokens, defines (hash or list), indents, and the source struct itself.
- **Side effects:** Frees heap; decrements `numtokens` via `PC_FreeToken`.
- **Calls:** `FreeScript`, `PC_FreeToken`, `PC_FreeDefine`, `FreeMemory`

### PC_ReadToken
- **Signature:** `int PC_ReadToken(source_t *source, token_t *token)`
- **Purpose:** Main token-read entry point. Handles `#` directives, `$` directives, adjacent string concatenation, conditional skip, and macro expansion.
- **Inputs:** Active source, output token buffer.
- **Outputs/Return:** `qtrue` if token read, `qfalse` at EOF or error.
- **Side effects:** Modifies `source->tokens` (pushes/pops), may expand macros back into source.
- **Calls:** `PC_ReadSourceToken`, `PC_ReadDirective`, `PC_ReadDollarDirective`, `PC_ExpandDefineIntoSource`, `PC_FindHashedDefine`, `PC_UnreadToken`

### PC_ExpandDefine
- **Signature:** `int PC_ExpandDefine(source_t *source, token_t *deftoken, define_t *define, token_t **firsttoken, token_t **lasttoken)`
- **Purpose:** Fully expands a macro invocation including argument substitution, `#` stringizing, and `##` token-merging.
- **Outputs/Return:** `qtrue` on success; sets `*firsttoken`/`*lasttoken` to the expanded token chain.
- **Side effects:** Allocates/frees tokens; reads parm tokens from source.
- **Calls:** `PC_ReadDefineParms`, `PC_CopyToken`, `PC_StringizeTokens`, `PC_MergeTokens`, `PC_FreeToken`, `PC_ExpandBuiltinDefine`

### PC_EvaluateTokens
- **Signature:** `int PC_EvaluateTokens(source_t *source, token_t *tokens, signed long int *intvalue, double *floatvalue, int integer)`
- **Purpose:** Evaluates a pre-collected token list as a constant expression using operator-precedence reduction (shunting-yard style with explicit doubly-linked operator/value heaps).
- **Outputs/Return:** `qtrue` + fills `*intvalue`/`*floatvalue`; `qfalse` on syntax/division error.
- **Notes:** Uses fixed-size stack arrays (`MAX_VALUES=64`, `MAX_OPERATORS=64`); supports ternary `?:`, `defined()`.

### PC_Directive_define
- **Signature:** `int PC_Directive_define(source_t *source)`
- **Purpose:** Parses a `#define` directive, builds a `define_t` with optional parameter list and token body, detects redefinition and recursion.
- **Calls:** `PC_ReadLine`, `PC_FindHashedDefine`, `PC_Directive_undef`, `PC_AddDefineToHash`, `PC_CopyToken`

### PC_AddGlobalDefine / PC_RemoveGlobalDefine / PC_RemoveAllGlobalDefines
- **Purpose:** Manage the `globaldefines` list that is cloned into every new source at load time.
- **Side effects:** Modify global `globaldefines` pointer.

### PC_LoadSourceHandle / PC_FreeSourceHandle / PC_ReadTokenHandle / PC_SourceFileAndLine
- **Purpose:** Handle-based public API used by `botlib_export_t` (see `botlib.h`). Maps integer handles to `sourceFiles[]` entries.
- **Notes:** Handle 0 is reserved/invalid; max 63 simultaneous open sources.

## Control Flow Notes
This file is not part of the game frame loop. It is invoked during **initialization** and **file loading** phases—specifically when the bot library loads character files, item/weapon configs, and chat scripts via `PC_LoadSourceHandle` → `PC_ReadTokenHandle`. `PC_ReadToken` drives the parse loop and is called externally until it returns `qfalse` (EOF).

## External Dependencies
- `l_script.h` — `script_t`, `token_t`, `PS_ReadToken`, `LoadScriptFile`, `LoadScriptMemory`, `FreeScript`, `EndOfScript`, `StripDoubleQuotes`, `PS_SetBaseFolder`
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `FreeMemory`
- `l_log.h` — `Log_Write` (used in hash-table debug print, conditionally)
- `be_interface.h` (BOTLIB) — `botimport.Print` for error/warning output
- `q_shared.h` — `Com_Memcpy`, `Com_Memset`, `Com_Error`, `Q_stricmp`
- `time.h` — `time()`, `ctime()` for `__DATE__`/`__TIME__` builtins
- `PC_NameHash`, `PC_AddDefineToHash`, `PC_FindHashedDefine` — defined in this file, used throughout
