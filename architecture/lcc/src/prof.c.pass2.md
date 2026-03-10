# lcc/src/prof.c — Enhanced Analysis

## Architectural Role

`prof.c` implements compiler-side instrumentation for basic-block and call-site profiling in the LCC C compiler. It is **not part of the Quake III engine runtime**; instead, it operates within the LCC compilation pipeline to inject profiling hooks into compiled code. When the `-b` (basic block counting) or `-a` (call profiling) flags are used, this module inserts data structures and function calls that will allow the compiled program to report execution coverage when run.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler main**: `prof_init()` is called during compiler initialization to register event handlers when `-b`, `-C`, or `-a` flags are detected
- **Event system**: Hooks registered via `attach()` (at `events.entry`, `events.returns`, `events.exit`, `events.end`, `events.calls`, `events.points`) are invoked during code generation for each function compiled

### Outgoing (what this file depends on)
- **Compiler symbol/IR generation**: `genident()`, `mksymbol()`, `array()`, `ptr()`, `ftype()` for creating synthetic symbols and types
- **Code generation interface**: `defglobal()`, `defpointer()`, `defconst()`, `space()` to emit profiling data sections
- **IR backend**: `IR->little_endian`, `IR->defconst()`, `IR->space()` for endianness-aware data emission
- **AST/tree operations**: `tree()`, `idtree()`, `vcall()`, `asgn()`, `incr()`, `rvalue()`, `walk()`, `consttree()` for tree manipulation
- **Type system**: `voidptype`, `inttype`, `charptype`, `unsignedtype` predefined types
- **Utility functions**: `append()`, `mkstr()`, `ltov()` for list/string operations
- **Global state**: `cfunc` (current function symbol during compilation)

## Design Patterns & Rationale

**Event-driven instrumentation injection**: Rather than modifying the compiler's core code generation, profiling inserts itself via an **event hook mechanism**. Each compilation event (`entry`, `return`, `exit`, etc.) triggers attached handlers that transform or inject code.

**Dual profiling modes**:
- **`-a` mode** (call profiling): Links each call site with `_caller` (external symbol) to trace the call graph dynamically from a `prof.out` file
- **`-b` mode** (basic block counting): Injects execution-point counters into a `_YYcounts` array and maps coordinates to source locations

**Lazy symbol creation**: `caller`, `prologue`, `epilogue` are created on first use (`static` initialization pattern), minimizing overhead for programs that don't use profiling.

**Endian-aware binary encoding**: Coordinates (file, line, column, index) are packed into a `union coordinate` with both little-endian and big-endian layouts, allowing the compiled binary to be portable across architectures.

## Data Flow Through This File

1. **Initialization** (`prof_init`): Parses command-line flags; if `-b` or `-a` detected, allocates `YYlink` and registers event handlers
2. **Function entry** (`bbentry`): Emits call to `_prologue(&afunc, &yylink)` at function start
3. **Function body**:
   - **Call sites** (`bbcall`): Wraps calls with `_caller` assignment to record call site metadata
   - **Execution points** (`bbincr`): Appends coordinate data to in-memory map; injects `yycounts[npoints++]++`
4. **Function exit** (`bbexit`): Emits call to `_epilogue(&afunc)`
5. **File end** (`bbvars`): Emits static data structures: file list, coordinate arrays, function list

## Learning Notes

**What a developer learns from this file:**
- How compiler instrumentation works at the AST level (tree transformation rather than text rewriting)
- Event-driven plugin architecture for extending compilation without monolithic patching
- Endianness-aware binary packing for cross-platform tool compatibility
- Separation of concerns: compiler generates data structures; external runtime library (`_prologue`, `_epilogue`, `_caller`) consumes them

**Idiomatic to this era/compiler:**
- **Bare metal tree construction** (`tree(RIGHT, ...)`) rather than builder APIs
- **Global state for compilation context** (`cfunc`, `events`) typical of 1990s-era compiler infrastructure
- **Symbol naming conventions** (YY-prefixed variables for compiler-generated symbols; underscore-prefixed for ABI boundaries)
- No type-safe AST visitor pattern; instead, direct tree field access and manual recursion

**Not inferable from code:**
- The semantics of individual event types (when `events.points` vs `events.calls` fires)
- Whether this profiling data is consumed by external tools or embedded in the binary
- Interaction with other LCC compilation phases

## Potential Issues

- **Silent overlap**: If both `-b` and `-C` (or `-a`) flags are used together, initialization sets up both basic-block and call profiling, but the interaction is not documented
- **Memory model assumption**: `maplist` and `filelist` are global and never cleared between compilations in a single compiler invocation; multi-file compilations would accumulate state
- **No overflow checks**: `npoints` counter increments without bounds; if a source file has >65536 execution points, the `y:16` bit-field would overflow silently
