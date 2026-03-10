# code/botlib/l_struct.c

## File Purpose
Provides generic serialization and deserialization of C structures to/from botlib script sources and plain text files. It maps a runtime `structdef_t` schema (field names, types, offsets) onto raw memory, enabling data-driven config loading and saving without hand-written parsers per struct.

## Core Responsibilities
- Look up a named field within a `fielddef_t` array
- Parse numeric values (int, char, float) from a `source_t` token stream with range validation
- Parse character literals and quoted strings from token streams
- Recursively deserialize a brace-delimited block from a `source_t` into a flat memory buffer
- Write indentation, float values (trailing-zero stripped), and full structures to a `FILE*`
- Recursively serialize a structure to an indented text file

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `fielddef_t` | struct (typedef) | Describes one field: name, byte offset, type flags, array size, float bounds, optional sub-struct pointer |
| `structdef_t` | struct (typedef) | Pairs a total `size` with a null-terminated `fielddef_t` array; serves as the schema for read/write |
| `source_t` | struct (typedef) | Pre-compiler source handle (defined in `l_precomp.h`); owns the token stream consumed during reading |
| `token_t` | struct (typedef) | Lexed token carrying type, subtype, `intvalue`, `floatvalue`, and `string` (from `l_script.h`) |

## Global / File-Static State
None.

## Key Functions / Methods

### FindField
- **Signature:** `fielddef_t *FindField(fielddef_t *defs, char *name)`
- **Purpose:** Linear search through a null-terminated `fielddef_t` array for a field matching `name`.
- **Inputs:** `defs` — field definition array; `name` — field name to find.
- **Outputs/Return:** Pointer to matching `fielddef_t`, or `NULL`.
- **Side effects:** None.
- **Calls:** `strcmp`
- **Notes:** Array termination detected by `defs[i].name == NULL`.

---

### ReadNumber
- **Signature:** `qboolean ReadNumber(source_t *source, fielddef_t *fd, void *p)`
- **Purpose:** Reads one numeric token (optionally preceded by `-`) and stores it into `p` as the type described by `fd`, enforcing type and optional bounds.
- **Inputs:** `source` — active token stream; `fd` — field descriptor; `p` — destination memory.
- **Outputs/Return:** `1` on success, `0` on parse or range error.
- **Side effects:** Emits `SourceError` on failure; writes through `p`.
- **Calls:** `PC_ExpectAnyToken`, `SourceError`, `Maximum`, `Minimum`
- **Notes:** Handles `FT_CHAR`, `FT_INT`, `FT_FLOAT`; `FT_UNSIGNED` restricts sign; `FT_BOUNDED` clamps allowed range using `fd->floatmin/floatmax`.

---

### ReadChar
- **Signature:** `qboolean ReadChar(source_t *source, fielddef_t *fd, void *p)`
- **Purpose:** Reads a single character — accepting either a single-quoted literal or a numeric value via `ReadNumber`.
- **Inputs/Outputs:** Same pattern as `ReadNumber`.
- **Side effects:** Writes one `char` through `p`.
- **Calls:** `PC_ExpectAnyToken`, `StripSingleQuotes`, `PC_UnreadLastToken`, `ReadNumber`

---

### ReadString
- **Signature:** `int ReadString(source_t *source, fielddef_t *fd, void *p)`
- **Purpose:** Reads a double-quoted string token and copies it (up to `MAX_STRINGFIELD` bytes) into `p`.
- **Inputs/Outputs:** As above; `p` receives a null-terminated C string.
- **Calls:** `PC_ExpectTokenType`, `StripDoubleQuotes`, `strncpy`
- **Notes:** Always null-terminates at `MAX_STRINGFIELD-1`.

---

### ReadStructure
- **Signature:** `int ReadStructure(source_t *source, structdef_t *def, char *structure)`
- **Purpose:** Deserializes a `{ field value ... }` block from `source` into the raw byte buffer `structure`, using `def` as the schema. Supports arrays (inner `{ v, v, ... }`) and recursive sub-structs.
- **Inputs:** `source` — token stream positioned before `{`; `def` — schema; `structure` — destination buffer.
- **Outputs/Return:** `qtrue` on success, `qfalse` on any parse error.
- **Side effects:** Calls `SourceError` on unknown fields or bad syntax.
- **Calls:** `PC_ExpectTokenString`, `PC_ExpectAnyToken`, `FindField`, `PC_CheckTokenString`, `ReadChar`, `ReadNumber`, `ReadString`, `ReadStructure` (recursive)
- **Notes:** Advances pointer `p` by the C sizeof each primitive after each element; sub-struct size comes from `fd->substruct->size`.

---

### WriteFloat
- **Signature:** `int WriteFloat(FILE *fp, float value)`
- **Purpose:** Writes a float to `fp` with trailing zeros (and trailing `.`) stripped.
- **Inputs:** `fp` — open file; `value` — float to write.
- **Outputs/Return:** `1` on success, `0` on `fprintf` failure.
- **Calls:** `sprintf`, `strlen`, `fprintf`

---

### WriteStructWithIndent
- **Signature:** `int WriteStructWithIndent(FILE *fp, structdef_t *def, char *structure, int indent)`
- **Purpose:** Recursively writes all fields of `structure` to `fp` in a human-readable, tab-indented `{ ... }` block matching the format `ReadStructure` expects.
- **Inputs:** `fp`, `def`, `structure`, `indent` — current indentation depth.
- **Outputs/Return:** `qtrue`/`qfalse`.
- **Side effects:** File I/O.
- **Calls:** `WriteIndent`, `fprintf`, `WriteFloat`, `WriteStructWithIndent` (recursive)
- **Notes:** Arrays emit `{v,v,...}` inline; sub-structs recurse with incremented indent.

---

### WriteStructure
- **Signature:** `int WriteStructure(FILE *fp, structdef_t *def, char *structure)`
- **Purpose:** Public entry point; delegates to `WriteStructWithIndent` at indent level 0.
- **Calls:** `WriteStructWithIndent`

## Control Flow Notes
This file has no frame or update participation. It is a pure utility library called at **load/init time** when the botlib reads configuration files (character files, weapon weights, item configs) via `ReadStructure`, and at **save time** when writing those configs back via `WriteStructure`. It sits below the precompiler (`l_precomp`) in the call stack and above any direct callers such as `be_ai_char.c`, `be_ai_weap.c`, and `be_ai_goal.c`.

## External Dependencies
- `l_precomp.h` — `source_t`, `PC_ExpectAnyToken`, `PC_ExpectTokenString`, `PC_ExpectTokenType`, `PC_CheckTokenString`, `PC_UnreadLastToken`, `SourceError`
- `l_script.h` — `token_t`, `StripDoubleQuotes`, `StripSingleQuotes`, `TT_*` constants
- `l_struct.h` — `fielddef_t`, `structdef_t`, `MAX_STRINGFIELD`, `FT_*` constants (self-header)
- `l_utils.h` — `Maximum`, `Minimum` macros (defined elsewhere)
- `q_shared.h` — `qboolean`, `qtrue`, `qfalse` (game shared types)
- `be_interface.h` — included for botlib import table context; no direct calls visible here
- Standard C: `strcmp`, `strncpy`, `sprintf`, `strlen`, `fprintf` — standard library
