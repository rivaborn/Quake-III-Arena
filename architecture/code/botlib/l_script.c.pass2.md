# code/botlib/l_script.c — Enhanced Analysis

## Architectural Role

This file is a **shared utility lexer** compiled into three separate build artifacts: botlib.so/DLL (runtime), BSPC.exe (offline compiler), and MEQCC (QuakeC compiler). It tokenizes arbitrary C-like script files—used by botlib for AAS metadata parsing, BSPC for geometry processing scripts, and internally by both to load configuration and data. The compile-time `#ifdef` multiplexing pattern (BOTLIB/BSPC/MEQCC/SCREWUP) decouples the tokenizer source from backend-specific I/O, logging, and memory allocation, allowing a single codebase to integrate seamlessly into three distinct execution contexts without runtime polymorphism overhead.

## Key Cross-References

### Incoming (who depends on this file)
- **botlib subsystem** (`code/botlib/be_aas_*.c`): Uses `LoadScriptFile`/`LoadScriptMemory` to parse AAS files (`*.aas`), entity BSP metadata, and possibly libvar configuration scripts  
- **BSPC tool** (`code/bspc/*.c`): Reuses the lexer for offline BSP→AAS compilation; shares AAS parsing pipeline with botlib
- **botlib internals** (`code/botlib/l_libvar.c`, `l_precomp.c`): Depend on `PS_ReadToken*` and `PS_ExpectToken*` for script parsing within the library

### Outgoing (what this file depends on)
- **Conditional backends** (via `#ifdef` blocks):
  - `BOTLIB`: Calls `botimport.FS_FOpenFile/FS_Read/FS_FCloseFile` for file I/O; `botimport.Print` for logging; `Com_Memset`, `Com_Memcpy`, `COM_Compress` from `qcommon/common.c`
  - `BSPC`/`MEQCC`: Use standard C `fopen/fread/fclose` and `printf`
- **Memory layer** (`l_memory.h`): `GetMemory`, `GetClearedMemory`, `FreeMemory`
- **Global `basefolder`** buffer: Runtime-mutable directory prefix used by `LoadScriptFile` to resolve relative paths

## Design Patterns & Rationale

1. **Compile-time Dependency Injection**: Instead of runtime vtables, the file includes different headers and uses conditional compilation to adapt to botlib, BSPC, or MEQCC environments. This avoids heap-allocated function pointers and indirection while maintaining zero coupling to any one backend.

2. **Punctuation Hash Table with Longest-First Ordering**: `PS_CreatePunctuationTable` builds a 256-slot hash table keyed on first byte, with entries linked by length descending. This enables greedy matching (e.g., `>>=` is tried before `>>`) without backtracking—a classic lexer optimization for multi-character operators.

3. **One-Token Pushback via `tokenavailable` Flag**: Rather than allocating a secondary token buffer, the parser provides `PS_UnreadLastToken` to set a flag and restore `script_p`, allowing lookahead without buffering overhead.

4. **Unified Buffer Allocation**: `LoadScriptFile` allocates a single contiguous block for both the `script_t` struct and the loaded file contents, minimizing heap fragmentation and simplifying lifetime management.

## Data Flow Through This File

1. **Intake**: `LoadScriptFile` or `LoadScriptMemory` receive a filename or buffer  
2. **Whitespace stripping**: `COM_Compress` removes comments and redundant whitespace from loaded buffer to reduce working set  
3. **Tokenization loop** (driven by caller):  
   - `PS_ReadToken` dispatches to type-specific readers (`PS_ReadString`, `PS_ReadNumber`, `PS_ReadName`, `PS_ReadPunctuation`)  
   - Each reader advances `script->script_p` and populates output `token_t`  
4. **Validation**: Caller uses `PS_ExpectToken*` / `PS_CheckToken*` to enforce grammar; errors logged via backend-specific channel  
5. **Exit**: `FreeScript` deallocates punctuation table and combined struct+buffer block

## Learning Notes

- **Multi-compilation idiom**: This pattern (unified source, multiple build targets) was common in pre-C++ game engines; modern engines use runtime polymorphism or separate codebases. The tradeoff here is zero runtime overhead vs. harder maintenance.
- **Whitespace compression as a preprocessing step**: Loading with `COM_Compress` trades I/O size and parse-time scans for slower file load. Suitable for development; shipped builds might preprocess scripts offline.
- **No state machine for nested contexts**: Unlike many production lexers, this does not track parenthesis depth or scope nesting. Callers (e.g., `l_libvar.c`, `l_precomp.c`) must implement their own grammar stack.
- **Escape sequence handling**: Supports C-standard escapes (`\n`, `\xHH`, `\DDD` decimal) but—notably—uses decimal, not octal, for numeric escapes, matching older Quake conventions.

## Potential Issues

1. **Thread-unsafe global `basefolder`**: If multiple threads load scripts concurrently from different directories, race conditions on `basefolder` reads will corrupt paths. Botlib itself may not be multithreaded, but the issue could surface during tools development or porting.

2. **No include-file support**: Unlike C preprocessors, the lexer cannot `#include` other scripts, forcing all logic into a single file or relying on callers to manually concatenate scripts—a limitation for large configuration hierarchies.

3. **Unbounded comment nesting untested**: Block comments (`/* */`) do not check for nesting; `/* /* inner */ */` will terminate early. Unlikely to cause issues in practice (not a Quake convention), but a subtle edge case.
