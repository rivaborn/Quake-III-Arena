# lcc/cpp/nlist.c — Enhanced Analysis

## Architectural Role

This file implements the **C preprocessor symbol table** for the LCC compiler, a build-time tool used to compile Quake III game code to QVM bytecode. It sits in the **offline compilation pipeline** (`lcc/cpp/`), separate from the runtime engine entirely. The symbol table here tracks preprocessor keywords and directives—establishing the lexical semantics that govern how game code (game VM, cgame VM, UI VM) is preprocessed before JIT/interpretation by the runtime `qcommon/vm.c` host.

## Key Cross-References

### Incoming (who depends on this file)
- Other `lcc/cpp/` preprocessor modules (e.g., `lex.c`, likely calls `lookup()` and `setup_kwtab()`)
- Main entry point in `lcc/cpp/cpp.c` or `lcc/etc/linux.c` (LCC driver code)

### Outgoing (what this file depends on)
- Memory allocators: `new()`, `newstring()` (LCC internal utilities)
- Token definitions: `Token` struct, `cp.h` header
- Character-set utilities: `quickset()` (for keyword prefix filtering)
- Platform: `getopt`, `optind`, `optarg` for CLI argument processing

**No direct runtime dependencies**: Unlike the engine subsystems (renderer, qcommon, etc.), this code is _never_ executed at runtime. Its output (compiled QVM bytecode) is consumed by the VM subsystem in `qcommon/vm_interpreted.c` or `vm_x86.c`, but this symbol table is discarded after compilation.

## Design Patterns & Rationale

- **Hash-table symbol table (NLSIZE=128)**: Open addressing with list chaining. Simple, cache-friendly for small keyword sets.
- **Pre-computed keyword table (`kwtab[]`)**: All preprocessor keywords (if, ifdef, define, etc.) hardcoded with token codes and semantic flags (`ISKW`, `ISMAC`, `ISUNCHANGE`). Avoids runtime string parsing.
- **Two-phase lookup**: `setup_kwtab()` initializes the table once at startup; `lookup()` performs amortized O(1) queries during tokenization.
- **Token struct design**: Uses length-prefixed tokens (`{t, len}`) instead of null-terminated strings—enables efficient substring comparison without scanning to null terminator.
- **Special `defined` handling**: The `defined` operator is marked `KDEFINED` but converted to `NAME` internally with a pre-baked `Tokenrow deftr` to inject its AST representation during preprocessing.

**Why this design?** Preprocessors must be fast; inlining keyword checks during lexing avoids a separate table lookup for every identifier. The flags (`ISUNCHANGE`, `ISDEFINED`, `ISMAC`) tag keywords that require special semantics (e.g., `__LINE__` cannot be undefined).

## Data Flow Through This File

1. **Init phase**: `setup_kwtab()` called once by LCC startup
   - Walks `kwtab[]` (hardcoded keyword list)
   - For each keyword: `lookup(&token, install=1)` inserts it into `nlist[]`
   - Sets semantic flags on each `Nlist` entry (e.g., `KDEFINE` → value, `ISKW` → flag)

2. **Preprocessing phase**: Lexer calls `lookup(&token, install=0)` for each identifier
   - Returns `Nlist*` if keyword, `NULL` if unknown
   - Lexer branches: if keyword → emit its token code (e.g., `KIF`, `KDEFINE`); else → treat as identifier

3. **Output**: Preprocessed token stream with directives resolved
   - Game code (game.c, cg_main.c, ui_main.c) → LCC → QVM bytecode → loaded by `qcommon/vm.c` at runtime

## Learning Notes

- **Era of the codebase**: Manual hash tables and static initialization typical of late-1990s compilers (LCC predates this release).
- **Contrast with modern engines**: Contemporary compilers use AST-based symbol tables with scope chains; this is purely lexical.
- **Determinism**: Keyword lookup is completely deterministic—no dynamic memory, no hash randomization—important for reproducible builds.
- **LCC's niche**: LCC was chosen for Quake III because it compiles to efficient bytecode for the VM sandbox. The symbol table here is part of that pipeline.
- **Unused surface**: `namebit[077+1]` (64 bytes) appears intended as a quick filter for character-to-token-type mapping, but usage is unclear from this file alone (likely checked by `quickset()`).

## Potential Issues

- **Fixed hash table size**: `NLSIZE=128` cannot grow. In a large codebase, collisions could degrade lookup to O(n). However, preprocessor keywords are finite (~20 here), so collision chain is short.
- **No resizing logic**: If `lookup(..., install=1)` is called with more than ~128 distinct identifiers, hash chains grow linearly. Not a practical issue for a small compiler.
- **Static `wd[128]` buffer**: Purpose unclear; likely a scratch buffer for preprocessing work, but scope is global (thread-unsafe if LCC ever multi-threaded).

---

**Summary**: This is a thin, fast preprocessor symbol table—a classic compiler artifact with no runtime relevance. Its role is purely to pipeline game source code through LCC into QVM bytecode consumed by the engine's VM subsystem.
