# code/botlib/l_script.h

## File Purpose
Defines the public interface for a lexicographical script parser used by the botlib. It provides token-based parsing of text scripts and configuration files, supporting C/C++-style syntax including strings, literals, numbers (decimal, hex, octal, binary, float), and a comprehensive punctuation set.

## Core Responsibilities
- Define token types and subtypes for lexical classification
- Define punctuation symbol constants (P_*) for all C/C++ operators and delimiters
- Declare the `script_t` state structure representing a loaded script with cursor tracking
- Declare the `token_t` structure for individual parsed tokens
- Declare the `punctuation_t` structure for customizable punctuation tables
- Expose the full parser API: read, expect, check, skip, unread operations
- Provide script lifecycle functions: load from file/memory, reset, free

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `punctuation_t` | struct | Linked-list node mapping a punctuation string to its integer ID |
| `token_t` | struct | Holds a parsed token: string, type, subtype, integer/float value, whitespace pointers, line info, chain pointer |
| `script_t` | struct | Full parser state: filename, buffer, read cursor (`script_p`), end pointer, line counters, token-unread flag, flags, punctuation table, last token, chain pointer |

## Global / File-Static State
None. (All state is encapsulated in `script_t` instances.)

## Key Functions / Methods

### PS_ReadToken
- Signature: `int PS_ReadToken(script_t *script, token_t *token)`
- Purpose: Read the next token from the script into `token`.
- Inputs: Active `script_t`, output `token_t *`
- Outputs/Return: Non-zero on success, 0 at end-of-script or error.
- Side effects: Advances `script->script_p`; updates `script->line`.
- Calls: Defined in `l_script.c`.
- Notes: Respects `SCFL_*` flags on the script.

### PS_ExpectTokenString
- Signature: `int PS_ExpectTokenString(script_t *script, char *string)`
- Purpose: Read next token and error if it doesn't match `string`.
- Inputs: Script, expected string literal.
- Outputs/Return: Non-zero on match, 0 with error on mismatch.
- Side effects: Calls `ScriptError` on mismatch.

### PS_ExpectTokenType
- Signature: `int PS_ExpectTokenType(script_t *script, int type, int subtype, token_t *token)`
- Purpose: Read next token and error if type/subtype don't match.
- Inputs: Script, expected `TT_*` type, expected subtype bitmask, output token.
- Outputs/Return: Non-zero on match.
- Side effects: Calls `ScriptError` on mismatch.

### PS_CheckTokenString
- Signature: `int PS_CheckTokenString(script_t *script, char *string)`
- Purpose: Peek/consume next token only if it matches `string`; no error if it doesn't.
- Outputs/Return: Non-zero if matched and consumed, 0 otherwise (token left unread).

### PS_UnreadLastToken / PS_UnreadToken
- Signature: `void PS_UnreadLastToken(script_t *script)` / `void PS_UnreadToken(script_t *script, token_t *token)`
- Purpose: Push back the last token so the next `PS_ReadToken` re-delivers it. `PS_UnreadLastToken` uses the script's internal last-token record; `PS_UnreadToken` uses a caller-supplied token.
- Side effects: Sets `script->tokenavailable`.

### LoadScriptFile / LoadScriptMemory
- Signature: `script_t *LoadScriptFile(const char *filename)` / `script_t *LoadScriptMemory(char *ptr, int length, char *name)`
- Purpose: Allocate and initialize a `script_t`, reading from disk or a memory buffer respectively.
- Outputs/Return: Heap-allocated `script_t *`, or NULL on failure.
- Side effects: File I/O (LoadScriptFile); heap allocation.

### ScriptError / ScriptWarning
- Signature: `void QDECL ScriptError(script_t *script, char *str, ...)` / `void QDECL ScriptWarning(script_t *script, char *str, ...)`
- Purpose: Variadic diagnostic printers that prepend filename and line number.
- Side effects: Console/log output; `ScriptError` may abort parsing depending on `SCFL_NOERRORS`.

### Notes
- `ReadSignedInt` / `ReadSignedFloat`: convenience wrappers that handle a leading `-` sign.
- `StripDoubleQuotes` / `StripSingleQuotes`: in-place string mutators for post-processing token strings.
- `SetScriptPunctuations`: replaces the active punctuation table; NULL resets to default C/C++ set.

## Control Flow Notes
This is a passive utility library invoked on demand. It participates in init-time parsing of bot configuration, AI weight files, character scripts, and AAS data — all loaded during botlib initialization (`BotLibSetup`). It is not called per-frame.

## External Dependencies
- No explicit includes shown; implementation (`l_script.c`) will pull in standard C I/O and string headers.
- `QDECL`: calling-convention macro defined in `q_shared.h` or conditionally as empty for BSPC builds.
- `BSPC`: build-time define selecting BSP compiler context over botlib context.
- `LoadScriptFile` depends on a file system abstraction defined elsewhere (likely `l_memory.h` / OS file I/O).
