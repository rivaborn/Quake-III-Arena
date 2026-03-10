# common/scriplib.c

## File Purpose
Implements a stack-based script/text tokenizer for the Quake III tools (q3map, bspc, q3radiant). It reads text files or in-memory buffers, tokenizes them token-by-token with support for `$include` directives, nested file inclusion, and structured matrix parsing/writing.

## Core Responsibilities
- Load script files from disk or parse directly from memory buffers
- Maintain a stack of up to 8 nested script contexts (for `$include` support)
- Skip whitespace, line/block comments (`;`, `#`, `//`, `/* */`)
- Tokenize input into the global `token[]` buffer (quoted and unquoted tokens)
- Track line numbers across file boundaries for error reporting
- Parse 1D/2D/3D float matrices from parenthesis-delimited token streams
- Write 1D/2D/3D float matrices back to a FILE in parenthesis-delimited format

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `script_t` | struct | Holds state for one script context: filename, buffer pointer, read cursor, end pointer, and current line number |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `scriptstack[MAX_INCLUDES]` | `script_t[8]` | global (file) | Fixed-size stack of nested script contexts |
| `script` | `script_t *` | global (file) | Pointer to the currently active script context |
| `scriptline` | `int` | global (exported) | Current line number, used in error messages |
| `token[MAXTOKEN]` | `char[1024]` | global (exported) | Buffer holding the most recently parsed token |
| `endofscript` | `qboolean` | global (exported) | Set to `qtrue` when all script input is exhausted |
| `tokenready` | `qboolean` | global (file) | Set by `UnGetToken`; causes next `GetToken` to replay current token |

## Key Functions / Methods

### AddScriptToStack
- **Signature:** `void AddScriptToStack( const char *filename )`
- **Purpose:** Pushes a new file onto the script inclusion stack and loads it into memory.
- **Inputs:** `filename` — path relative to the current script search path.
- **Outputs/Return:** void
- **Side effects:** Increments `script` pointer; calls `LoadFile` (heap alloc); prints "entering \<filename\>" to stdout.
- **Calls:** `ExpandPath`, `LoadFile`, `Error`, `printf`
- **Notes:** Errors fatally if the stack depth exceeds `MAX_INCLUDES` (8).

### LoadScriptFile
- **Signature:** `void LoadScriptFile( const char *filename )`
- **Purpose:** Initializes the script stack to depth 1 and loads a file as the root script.
- **Inputs:** `filename` — path to the script file.
- **Outputs/Return:** void
- **Side effects:** Resets `script` to `scriptstack`; clears `endofscript` and `tokenready`.
- **Calls:** `AddScriptToStack`
- **Notes:** Must be called before any `GetToken` calls for a new parse session.

### ParseFromMemory
- **Signature:** `void ParseFromMemory( char *buffer, int size )`
- **Purpose:** Initializes parsing from a caller-supplied in-memory buffer instead of a file.
- **Inputs:** `buffer` — pointer to text data; `size` — byte length.
- **Outputs/Return:** void
- **Side effects:** Pushes one context onto the stack without allocating; sets filename to `"memory buffer"`; clears `endofscript`/`tokenready`.
- **Calls:** `Error`
- **Notes:** The buffer is not freed on `EndOfScript` (only heap-allocated file buffers are freed).

### UnGetToken
- **Signature:** `void UnGetToken( void )`
- **Purpose:** Pushes back the current token so the next `GetToken` re-returns it.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Sets `tokenready = qtrue`.
- **Notes:** Only one token of look-ahead is supported. Pushing back across a line boundary is possible, which may affect crossline-checking on re-read.

### EndOfScript
- **Signature:** `qboolean EndOfScript( qboolean crossline )`
- **Purpose:** Handles end-of-buffer: either terminates parsing or pops the include stack and continues.
- **Inputs:** `crossline` — if false, triggers a fatal error (incomplete line).
- **Outputs/Return:** `qfalse` on true end-of-input; otherwise delegates to `GetToken`.
- **Side effects:** `free`s the current script buffer (unless memory buffer); decrements `script`; updates `scriptline`; prints "returning to \<filename\>".
- **Calls:** `Error`, `free`, `GetToken`, `printf`

### GetToken
- **Signature:** `qboolean GetToken( qboolean crossline )`
- **Purpose:** Primary tokenizer. Advances the cursor, skips whitespace/comments, and fills `token[]` with the next token.
- **Inputs:** `crossline` — if `qfalse`, newlines and comments cause a fatal error (token must be on current line).
- **Outputs/Return:** `qtrue` if a token was read; `qfalse` at end of all input.
- **Side effects:** Modifies `token[]`, `scriptline`, `script->script_p`, `script->line`. Handles `$include` by calling `AddScriptToStack` recursively.
- **Calls:** `EndOfScript`, `AddScriptToStack`, `Error`
- **Notes:** Bug risk: the `/* */` comment termination check uses `&&` instead of `||` — it requires both `*` and `/` to be absent simultaneously rather than detecting `*/` correctly.

### TokenAvailable
- **Signature:** `qboolean TokenAvailable( void )`
- **Purpose:** Peeks ahead to determine if a token remains on the current line.
- **Inputs:** none
- **Outputs/Return:** `qtrue` if the next token is on the same line; `qfalse` otherwise.
- **Side effects:** Calls `GetToken`/`UnGetToken` internally (no net state change).
- **Calls:** `GetToken`, `UnGetToken`

### MatchToken
- **Signature:** `void MatchToken( char *match )`
- **Purpose:** Asserts the next token equals `match`, fatally errors if not.
- **Calls:** `GetToken`, `Error`

### Parse1DMatrix / Parse2DMatrix / Parse3DMatrix
- **Signature:** `void ParseNDMatrix( int dims..., vec_t *m )`
- **Purpose:** Parse parenthesis-wrapped float vectors/matrices from the token stream into a flat array.
- **Notes:** `Parse2DMatrix` and `Parse3DMatrix` recurse into lower-dimension parsers. Expect `(` … `)` delimiters at each level.

### Write1DMatrix / Write2DMatrix / Write3DMatrix
- **Signature:** `void WriteNDMatrix( FILE *f, int dims..., vec_t *m )`
- **Purpose:** Serialize flat float arrays as parenthesis-wrapped ASCII to a FILE.
- **Notes:** Integers are written without a decimal point for readability.

## Control Flow Notes
This file is a **tool-time** (offline) utility used by map compilers (`q3map`, `bspc`) and the editor (`q3radiant`). It has no role in the runtime game engine frame loop. It is invoked during tool startup when config or map script files are loaded, and at any point parsing is needed. It is not used by the game, cgame, or renderer at runtime.

## External Dependencies
- **`cmdlib.h`** — `Error`, `LoadFile`, `ExpandPath`, `qboolean`, `qtrue`/`qfalse`
- **`scriplib.h`** — declares all exported symbols; also pulls in `mathlib.h` for `vec_t`
- Standard C: `stdio.h` (FILE, printf, fprintf), `stdlib.h` (free), `string.h` (strcmp, strcpy), `atof`
