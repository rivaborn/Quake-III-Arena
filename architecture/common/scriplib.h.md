# common/scriplib.h

## File Purpose
Public interface header for the tool-suite script/token parser used by offline map-compilation tools (q3map, bspc, q3radiant). It exposes a simple line-oriented tokenizer and matrix I/O helpers built on top of a single global parse cursor.

## Core Responsibilities
- Declare global state for the active parse cursor (`scriptbuffer`, `script_p`, `scriptend_p`, `token`, etc.)
- Expose file-based and memory-based script loading (`LoadScriptFile`, `ParseFromMemory`)
- Provide token-stream control: fetch, un-fetch, and lookahead (`GetToken`, `UnGetToken`, `TokenAvailable`)
- Expose exact-match token assertion (`MatchToken`)
- Provide 1-D, 2-D, and 3-D float matrix parsing from the token stream
- Provide symmetric 1-D, 2-D, and 3-D float matrix writing to a `FILE *`

## Key Types / Data Structures

None. (All types are imported from `cmdlib.h` / `mathlib.h`.)

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `token` | `char[MAXTOKEN]` | global (extern) | Holds the most-recently fetched token string |
| `scriptbuffer` | `char *` | global (extern) | Base pointer of the loaded script text |
| `script_p` | `char *` | global (extern) | Current read position within the script buffer |
| `scriptend_p` | `char *` | global (extern) | One-past-end sentinel of the script buffer |
| `grabbed` | `int` | global (extern) | Count of tokens grabbed (diagnostic / bookkeeping) |
| `scriptline` | `int` | global (extern) | Current line number in the source script (for error messages) |
| `endofscript` | `qboolean` | global (extern) | Set when `script_p` reaches `scriptend_p` |

## Key Functions / Methods

### LoadScriptFile
- **Signature:** `void LoadScriptFile( const char *filename );`
- **Purpose:** Loads a named text file into `scriptbuffer` and resets all parse-cursor globals.
- **Inputs:** `filename` — path to the script/map/shader text file.
- **Outputs/Return:** void; side-effects only.
- **Side effects:** Allocates `scriptbuffer`; sets `script_p`, `scriptend_p`, `scriptline = 1`, `endofscript = qfalse`.
- **Calls:** Likely `LoadFile` (from `cmdlib`).
- **Notes:** Replaces any previously loaded script; no stack — only one script active at a time.

### ParseFromMemory
- **Signature:** `void ParseFromMemory( char *buffer, int size );`
- **Purpose:** Points the parse cursor at a caller-supplied in-memory buffer instead of a file.
- **Inputs:** `buffer` — pointer to text data; `size` — byte length.
- **Outputs/Return:** void.
- **Side effects:** Sets `scriptbuffer`, `script_p`, `scriptend_p`, `scriptline`, `endofscript`; does **not** take ownership of `buffer`.
- **Calls:** Not inferable from this file.
- **Notes:** Allows parsing of embedded or procedurally generated text without a disk file.

### GetToken
- **Signature:** `qboolean GetToken( qboolean crossline );`
- **Purpose:** Advances `script_p` past whitespace/comments and copies the next token into `token[]`.
- **Inputs:** `crossline` — if `qfalse`, returns `qfalse` (or errors) when a newline is encountered before a token.
- **Outputs/Return:** `qtrue` if a token was consumed; `qfalse` at end-of-script or denied cross-line.
- **Side effects:** Mutates `token`, `script_p`, `scriptline`, `grabbed`, `endofscript`.
- **Calls:** Not inferable from this file.
- **Notes:** Primary tokenizer entry point; callers must check return value or use `TokenAvailable` first.

### UnGetToken
- **Signature:** `void UnGetToken( void );`
- **Purpose:** Pushes the last token back so the next `GetToken` re-returns it (single-token lookahead).
- **Inputs:** None.
- **Outputs/Return:** void.
- **Side effects:** Rewinds `script_p` or sets an internal un-get flag (implementation detail).
- **Calls:** Not inferable from this file.
- **Notes:** Only one level of un-get is supported.

### TokenAvailable
- **Signature:** `qboolean TokenAvailable( void );`
- **Purpose:** Non-destructive peek — returns `qtrue` if a token exists on the current line.
- **Inputs:** None.
- **Outputs/Return:** `qtrue` / `qfalse`.
- **Side effects:** None (read-only scan).
- **Notes:** Used to optionally parse fields without consuming mandatory tokens.

### MatchToken
- **Signature:** `void MatchToken( char *match );`
- **Purpose:** Calls `GetToken` and aborts with an error if the result does not equal `match`.
- **Inputs:** `match` — expected token string.
- **Outputs/Return:** void.
- **Side effects:** Advances parse cursor; calls `Error` on mismatch.
- **Notes:** Convenience assertion for structured formats (e.g., `{`, `}`).

### Parse1DMatrix / Parse2DMatrix / Parse3DMatrix
- **Signature:** `void Parse1DMatrix( int x, vec_t *m );` / `Parse2DMatrix( int y, int x, vec_t *m );` / `Parse3DMatrix( int z, int y, int x, vec_t *m );`
- **Purpose:** Parse a parenthesis-delimited matrix of floats from the token stream into a flat `vec_t` array.
- **Inputs:** Dimension sizes; `m` — output buffer (caller-allocated, row-major).
- **Outputs/Return:** void; fills `m`.
- **Side effects:** Consumes tokens; calls `MatchToken` for `(` / `)` delimiters.
- **Notes:** 2-D and 3-D variants call the lower-dimension helpers recursively.

### Write1DMatrix / Write2DMatrix / Write3DMatrix
- **Signature:** `void Write1DMatrix( FILE *f, int x, vec_t *m );` / etc.
- **Purpose:** Serialize a flat `vec_t` matrix to a `FILE *` in the same parenthesized text format that the Parse* functions read.
- **Inputs:** `f` — open file handle; dimension sizes; `m` — source data.
- **Outputs/Return:** void.
- **Side effects:** Writes bytes to `f`.
- **Notes:** Symmetric counterpart to the Parse* family; used when writing BSP/map data back to disk.

## Control Flow Notes
This header is **not** part of the runtime game engine. It is used exclusively during the **offline build pipeline** (map compilation, BSP generation, radiant editor). There is no frame/update loop; parsing is sequential, driven by tool `main()` functions reading `.map`, `.shader`, or config files top-to-bottom.

## External Dependencies
- `common/cmdlib.h` — `qboolean`, `LoadFile`, `Error`, file utilities
- `common/mathlib.h` — `vec_t`, `vec3_t` (float/double scalar type used by matrix helpers)
- `<stdio.h>` (via `cmdlib.h`) — `FILE *` used by Write* functions
