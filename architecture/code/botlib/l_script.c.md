# code/botlib/l_script.c

## File Purpose
Implements a reusable lexicographical (lexer/tokenizer) parser used by the Quake III bot library, BSP converter (BSPC), and MrElusive's QuakeC Compiler. It parses C-like script files into typed tokens (strings, numbers, names, punctuation) from either file or memory buffers.

## Core Responsibilities
- Load script text from disk (`LoadScriptFile`) or memory (`LoadScriptMemory`) into a `script_t` context
- Advance through whitespace and C/C++-style comments (`PS_ReadWhiteSpace`)
- Tokenize input into strings, literals, numbers (decimal/hex/octal/binary), identifiers, and punctuation
- Provide expect/check helpers for parser consumers to assert or conditionally consume tokens
- Support token unread (one-token pushback via `script->tokenavailable`)
- Route error/warning output to the correct backend (botlib, MEQCC, BSPC) via compile-time `#ifdef`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `punctuation_t` | struct | Linked-list node holding a punctuation string, numeric ID, and `next` pointer for hash-chained lookup |
| `token_t` | struct | Holds a parsed token: string value, type, subtype flags, optional int/float values, whitespace pointers, line info |
| `script_t` | struct | Complete parser state: buffer pointers, line counter, token pushback slot, punctuation table, flags, filename |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `default_punctuations[]` | `punctuation_t[]` | File-static (global linkage) | Default C/C++ punctuation set, ordered longest-first; used when no custom set is provided |
| `basefolder` | `char[]` | Global | Base directory prefix prepended to filenames on `LoadScriptFile` |

## Key Functions / Methods

### PS_CreatePunctuationTable
- **Signature:** `void PS_CreatePunctuationTable(script_t *script, punctuation_t *punctuations)`
- **Purpose:** Builds a 256-entry hash table (keyed on first byte of punctuation string) with entries sorted longest-first for greedy matching.
- **Inputs:** Script context; array of `punctuation_t` terminated by `{NULL,0}`.
- **Outputs/Return:** Void; populates `script->punctuationtable`.
- **Side effects:** Allocates 256-pointer array via `GetMemory` if not already allocated; mutates `next` pointers in the passed punctuation array.
- **Calls:** `GetMemory`, `Com_Memset`, `strlen`.

### PS_ReadWhiteSpace
- **Signature:** `int PS_ReadWhiteSpace(script_t *script)`
- **Purpose:** Skips blanks, tabs, newlines, `//` line comments, and `/* */` block comments; increments `script->line` on each `\n`.
- **Inputs:** Script context with `script_p` positioned at current character.
- **Outputs/Return:** 1 if non-whitespace found; 0 at EOF.
- **Side effects:** Advances `script->script_p`; modifies `script->line`.

### PS_ReadToken
- **Signature:** `int PS_ReadToken(script_t *script, token_t *token)`
- **Purpose:** Main tokenizer dispatch. Returns pushed-back token if available, otherwise skips whitespace and delegates to the appropriate typed reader based on the leading character.
- **Inputs:** Script context, output token buffer.
- **Outputs/Return:** 1 on success, 0 at EOF or error.
- **Side effects:** Updates `script->lastscript_p`, `script->lastline`, `script->whitespace_p`, `script->endwhitespace_p`, `script->token`; calls `PS_ReadWhiteSpace`, `PS_ReadString`, `PS_ReadNumber`, `PS_ReadName`, `PS_ReadPunctuation`, or `PS_ReadPrimitive`.

### LoadScriptFile
- **Signature:** `script_t *LoadScriptFile(const char *filename)`
- **Purpose:** Allocates a combined `script_t + buffer` block, reads file contents into it, then compresses whitespace via `COM_Compress`.
- **Inputs:** Filename string; uses `basefolder` prefix when non-empty.
- **Outputs/Return:** Initialized `script_t *`; `NULL` on failure.
- **Side effects:** File I/O (botlib: `botimport.FS_FOpenFile`/`FS_Read`/`FS_FCloseFile`; else `fopen`/`fread`/`fclose`); heap allocation via `GetClearedMemory`.
- **Calls:** `SetScriptPunctuations`, `COM_Compress`, `Com_Memset`.

### LoadScriptMemory
- **Signature:** `script_t *LoadScriptMemory(char *ptr, int length, char *name)`
- **Purpose:** Same initialization as `LoadScriptFile` but copies from a caller-supplied memory buffer instead of disk.
- **Side effects:** Heap allocation; `Com_Memcpy` of source buffer.

### FreeScript
- **Signature:** `void FreeScript(script_t *script)`
- **Purpose:** Frees punctuation table (if allocated) and the combined script/buffer allocation.
- **Side effects:** `FreeMemory` calls.

### PS_ExpectTokenString / PS_ExpectTokenType / PS_ExpectAnyToken
- Consume and validate the next token; call `ScriptError` and return 0 on mismatch. Used by higher-level parsers to assert grammar.

### PS_CheckTokenString / PS_CheckTokenType
- Non-consuming peek: reads a token, returns 1 and keeps it consumed on match; restores `script_p` to `lastscript_p` on mismatch (one-token rewind).

### Notes
- `NumberValue`: converts token string to `intvalue`/`floatvalue` based on subtype flags; only compiled when `NUMBERVALUE` is defined.
- `ReadSignedFloat`, `ReadSignedInt`: convenience helpers that handle an optional leading `-` sign.
- `StripDoubleQuotes`, `StripSingleQuotes`: in-place string mutators.
- `PS_UnreadLastToken` / `PS_UnreadToken`: set `tokenavailable` flag for one-token pushback.
- `ScriptSkipTo`: raw character-level scan to a target string, bypassing tokenization.

## Control Flow Notes
This file is a **utility library** with no frame or update loop. It is initialized on demand when a consumer calls `LoadScriptFile` or `LoadScriptMemory`, then driven entirely by caller-issued `PS_ReadToken` / `PS_ExpectToken*` calls. `FreeScript` is the shutdown path. It does not register itself with the engine's init/frame/shutdown cycle.

## External Dependencies
- `q_shared.h` — `Com_Memset`, `Com_Memcpy`, `Com_sprintf`, `COM_Compress`, `qboolean`, `MAX_QPATH`, file handle types (BOTLIB build)
- `botlib.h` / `be_interface.h` — `botimport` (for `Print`, `FS_FOpenFile`, `FS_Read`, `FS_FCloseFile`) (BOTLIB build)
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `FreeMemory` (defined in `l_memory.c`)
- `l_log.h` — `Log_Print` (BSPC build only)
- `COM_Compress` — defined in `qcommon/common.c` (or equivalent); strips comments/redundant whitespace from loaded buffer
