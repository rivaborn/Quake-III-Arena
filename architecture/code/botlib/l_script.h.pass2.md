# code/botlib/l_script.h — Enhanced Analysis

## Architectural Role

This file is the **lexical foundation of the botlib utility layer**, providing the token-based parser infrastructure used throughout botlib for loading and parsing diverse configuration and data formats. Rather than depending on the engine's file system, botlib is designed as a self-contained library with its own parsing stack (this lexer + `l_precomp.c` preprocessor + `l_struct.c` struct serialization + `l_libvar.c` variable system). The script parser tokenizes AAS binary file headers, AI weight config, character definitions, and procedural script formats used by the `be_ai_*.c` modules and offline `bspc` compiler.

## Key Cross-References

### Incoming (who depends on this file)
- **code/botlib/l_precomp.c** — Preprocessor builds atop the lexer, consuming its token stream to handle `#include`, `#define`, and macro expansion
- **code/botlib/l_libvar.c** — Variable system parses variable declarations and assignments using token-by-token reading
- **code/botlib/l_struct.c** — Struct serialization parses struct field definitions and binary data layouts
- **code/botlib/be_aas_file.c** — Loads and parses AAS binary file metadata and routing caches (uses script parser for header/cache format)
- **code/botlib/be_ai_weight.c, be_ai_char.c, be_ai_chat.c, be_ai_gen.c** — All AI subsystems parse fuzzy logic weight files, character templates, and chat scripts
- **code/bspc/be_aas_bspc.c** — Offline BSP→AAS compiler reuses botlib's complete parsing stack to read map data and generate AAS

### Outgoing (what this file depends on)
- **code/qcommon or OS file I/O** — `LoadScriptFile` delegates file reading to filesystem (declared here, implemented elsewhere; uses `botlib_import_t` or native I/O)
- **Standard C library** — `malloc`/`free` via `l_memory.c`, `strcmp`, `strlen`, etc.
- **q_shared.h** — `QDECL` calling convention macro; foundational types

## Design Patterns & Rationale

**Hand-Written Lexer with Customizable Syntax**  
Rather than a generated lexer (lex/yacc), this is a hand-coded scanner optimized for botlib's self-contained design. The `SetScriptPunctuations` API allows swapping punctuation tables at runtime—enabling the same lexer to tokenize C-like code (AAS data), Quake console scripts, and AI weight syntax without modification.

**Token Lookahead via Unread**  
The `PS_UnreadLastToken` / `PS_UnreadToken` pattern allows single-token lookahead without copying; callers can read speculatively, check type/value, and "push back" for re-parsing. This avoids token buffering overhead while supporting recursive-descent parsing style.

**Whitespace Preservation for Preprocessing**  
Unlike traditional lexers that discard whitespace, `script_t` tracks `whitespace_p` and `endwhitespace_p` separately. This design supports the preprocessor (`l_precomp.c`), which must preserve formatting and handle comments and macros without destroying source structure—a critical capability for offline tools like bspc.

**Flags for Parsing Variants**  
The `SCFL_*` flags (e.g., `SCFL_NOSTRINGESCAPECHARS`, `SCFL_NOBINARYNUMBERS`) allow a single implementation to serve multiple data formats. For example, AAS binary format may not need binary number literals, while C-like code does.

## Data Flow Through This File

1. **Initialization**: `LoadScriptFile` or `LoadScriptMemory` allocates a `script_t`, reads buffer, initializes cursors and line tracking.
2. **Tokenization**: Consumers call `PS_ReadToken` in a loop; lexer scans from `script_p`, classifies token (string, number, name, punctuation), populates `token_t`, advances `script_p`, tracks `line`.
3. **Lookahead & Error Recovery**: Callers use `PS_CheckTokenString` / `PS_ExpectTokenType` to validate grammar; on mismatch, `ScriptError` is invoked with file/line context.
4. **Token Pushback**: `PS_UnreadLastToken` resets parsing state so next `PS_ReadToken` re-delivers the last token—used by recursive-descent parsers to backtrack without token buffering.
5. **Cleanup**: `FreeScript` deallocates the buffer and token chain.

The flow is **synchronous and single-threaded**: each parse is a sequential scan, with no concurrency or async I/O.

## Learning Notes

**Era-Appropriate Design (Early 2000s)**  
Quake III's lexer reflects pre-STL, pre-Boost C practices: manual buffer management, fixed limits (`MAX_TOKEN = 1024`), and opaque pointer-based data structures. Modern engines would use lexer generators (flex), regex libraries, or third-party parsers (ANTLR, Tree-sitter).

**Self-Contained Library Pattern**  
Botlib's inclusion of a complete parsing stack (lexer, preprocessor, struct serializer, variable system) was intentional—enabling it to be compiled standalone for offline tools (`bspc`, `q3map`) without engine dependencies. This contrasts with modern modular designs that isolate I/O and parsing behind pluggable interfaces.

**Number Parsing Sophistication**  
The support for multiple number bases (decimal, hex, octal, binary) and types (integer, long, float, unsigned) shows this lexer was designed for C-like syntax, not just simple configuration files. The `TT_FLOAT` / `TT_INTEGER` / `TT_UNSIGNED` subtypes allow consumers to enforce type constraints (e.g., "this field must be unsigned").

**Punctuation as Pluggable Strategy**  
The `punctuation_t` linked list and `SetScriptPunctuations` function allow swapping operator/delimiter vocabularies without recompilation. This is a simple but effective strategy pattern, enabling the same lexer to tokenize different "languages."

## Potential Issues

- **Token Buffer Overflow**: `MAX_TOKEN` is fixed at 1024 characters. Malformed input with unterminated strings could overflow `token.string[]`. Implementation should bounds-check; callers should validate token lengths.

- **Whitespace Pointer Lifetime**: If a caller retains `token->whitespace_p` and later frees the `script_t`, the pointer dangles. The API doesn't document ownership; callers must either copy whitespace or guarantee script lifetime exceeds token lifetime.

- **Single-Unread Limitation**: The `tokenavailable` flag is a boolean, allowing only one level of pushback. Code that calls `PS_UnreadLastToken` twice without a read in between will silently discard the first unread. No protection against this misuse.

- **Line Tracking Overhead**: Scanning every character for `\n` adds per-token cost. For very large scripts, a byte-offset index into a line table could be faster, though this design prioritizes simplicity and accuracy.

- **Customization Brittleness**: Changing punctuation tables mid-parse could cause token stream discontinuities if the lexer has cached punctuation pointers. The API doesn't forbid this, but it's error-prone.
