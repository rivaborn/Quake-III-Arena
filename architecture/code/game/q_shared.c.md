# code/game/q_shared.c

## File Purpose
A stateless utility library compiled into every Quake III code module (game, cgame, ui, botlib). It provides portable string handling, text parsing, byte-order swapping, formatted output, and info-string manipulation that must be available in all execution environments including the QVM.

## Core Responsibilities
- Clamping, path, and file extension utilities
- Byte-order swap primitives for cross-platform endianness handling
- Tokenizing text parser with comment stripping and line tracking
- Safe string library replacements (`Q_str*`, `Q_strncpyz`, etc.)
- Color-sequence-aware string utilities (`Q_PrintStrlen`, `Q_CleanStr`)
- `va()` / `Com_sprintf()` formatted print helpers
- Info-string key/value encoding, lookup, insertion, and removal

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `_FloatByteUnion` | typedef (union) | Punning between `float` and `unsigned int` for byte-swap |
| `qint64` | typedef (struct, from header) | 8-byte integer represented as bytes for QVM compatibility |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `com_token` | `char[MAX_TOKEN_CHARS]` | static | Scratch buffer returned by `COM_ParseExt` |
| `com_parsename` | `char[MAX_TOKEN_CHARS]` | static | Source name for parse error messages |
| `com_lines` | `int` | static | Current line counter, incremented on `\n` during parsing |

## Key Functions / Methods

### Com_Clamp
- Signature: `float Com_Clamp(float min, float max, float value)`
- Purpose: Clamps a float to `[min, max]`.
- Inputs: bounds and value.
- Outputs/Return: clamped float.
- Side effects: None.
- Calls: None.

### COM_ParseExt
- Signature: `char *COM_ParseExt(char **data_p, qboolean allowLineBreaks)`
- Purpose: Core tokenizer. Extracts the next whitespace-delimited token or quoted string from `*data_p`, skipping `//` and `/* */` comments.
- Inputs: pointer-to-pointer into source text; flag controlling newline behavior.
- Outputs/Return: pointer to static `com_token`; advances `*data_p`.
- Side effects: Writes `com_token`; increments `com_lines`.
- Calls: `SkipWhitespace`.
- Notes: Returns empty string (never NULL). Tokens exceeding `MAX_TOKEN_CHARS` are silently discarded (len reset to 0).

### COM_Compress
- Signature: `int COM_Compress(char *data_p)`
- Purpose: In-place strips `//` and `/* */` comments, collapses whitespace, preserves quoted strings and single newlines.
- Inputs: mutable string buffer.
- Outputs/Return: length of compressed result.
- Side effects: Overwrites input buffer.
- Calls: Nothing.

### Q_strncpyz
- Signature: `void Q_strncpyz(char *dest, const char *src, int destsize)`
- Purpose: `strncpy` wrapper that always NUL-terminates and validates arguments.
- Inputs: destination buffer, source string, destination size.
- Side effects: Calls `Com_Error(ERR_FATAL, ...)` on NULL or zero-size arguments.
- Calls: `strncpy`, `Com_Error`.

### Com_sprintf
- Signature: `void QDECL Com_sprintf(char *dest, int size, const char *fmt, ...)`
- Purpose: `vsprintf` into a 32 KB stack buffer then copies into `dest` with size clamping.
- Side effects: Calls `Com_Error` on buffer overflow; logs on dest overflow; x86 debug break via inline asm.
- Calls: `vsprintf`, `Com_Error`, `Com_Printf`, `Q_strncpyz`.
- Notes: Stack buffer is 32000 bytes; hard error if `vsprintf` exceeds it.

### va
- Signature: `char * QDECL va(char *format, ...)`
- Purpose: Varargs `sprintf` into one of two 32 KB alternating static buffers.
- Outputs/Return: pointer to filled static buffer.
- Side effects: Modifies `string[0]` or `string[1]`; advances static `index`.
- Notes: Not re-entrant beyond 2 nesting levels.

### Info_ValueForKey
- Signature: `char *Info_ValueForKey(const char *s, const char *key)`
- Purpose: Searches a `\key\value\key\value` info string and returns the value for `key`.
- Outputs/Return: pointer to one of two alternating static value buffers, or `""`.
- Side effects: `Com_Error(ERR_DROP)` on oversized string; flips `valueindex` on each call.
- Calls: `Q_stricmp`, `Com_Error`.

### Info_SetValueForKey / Info_SetValueForKey_Big
- Signature: `void Info_SetValueForKey(char *s, const char *key, const char *value)`
- Purpose: Inserts or replaces a key/value pair in an info string; validates against forbidden characters (`\`, `;`, `"`).
- Side effects: Modifies `s` in place; calls `Com_Error` on oversize, `Com_Printf` on invalid chars or length exceeded.
- Calls: `Info_RemoveKey`, `Com_sprintf`, `Com_Error`, `Com_Printf`.

### Notes (minor helpers)
- `ShortSwap`, `LongSwap`, `Long64Swap`, `FloatSwap` — byte-reversal primitives; `*NoSwap` variants are identity passthroughs.
- `COM_SkipPath`, `COM_StripExtension`, `COM_DefaultExtension` — path string utilities.
- `SkipBracedSection`, `SkipRestOfLine` — parser navigation helpers.
- `Parse1DMatrix` / `Parse2DMatrix` / `Parse3DMatrix` — read parenthesis-delimited float arrays via `COM_Parse`.
- `Q_isprint/islower/isupper/isalpha/strrchr/stricmp/strncmp/stricmpn/strlwr/strupr/strcat/PrintStrlen/CleanStr` — portable CRT replacements.

## Control Flow Notes
This file has no init/frame/shutdown participation. It is a pure utility library. It is `#include`d (or linked) into every module at compile time. `COM_BeginParseSession` should be called before any `COM_Parse*` calls to set the source name and reset `com_lines`. The byte-swap functions are called directly (the old `Swap_Init` function-pointer dispatch system is commented out).

## External Dependencies
- `#include "q_shared.h"` — all type definitions, macros, and prototypes.
- `Com_Error`, `Com_Printf` — defined in `qcommon/common.c` (host side) or provided via syscall trap in VM modules.
- Standard C: `vsprintf`, `strncpy`, `strlen`, `strchr`, `strcmp`, `strcpy`, `strcat`, `atof`, `tolower`, `toupper`.
