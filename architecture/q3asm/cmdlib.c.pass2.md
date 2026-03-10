# q3asm/cmdlib.c — Enhanced Analysis

## Architectural Role

This file is a **shared utility foundation for offline tools** in the Quake III Arena build pipeline (Q3ASM, Q3Map, BSPC, Radiant). It provides cross-platform abstraction, command-line argument handling, path canonicalization within the engine's directory convention (basedir/gamedir split), and basic I/O and parsing utilities that all tools depend on. Unlike the runtime engine (which is compartmentalized by subsystem), the tool ecosystem treats `cmdlib` as a central dependency layer—no tool is standalone without it.

## Key Cross-References

### Incoming (who depends on this file)
- **Tool modules across `q3asm/`, `q3map/`, `code/bspc/`, `code/common/`**: All consume `cmdlib.h` for `Error()`, `ExpandPath()`, `ExpandGamePath()`, `COM_Parse()`, `copystring()`, path canonicalization
- **`code/client` and `code/server` (runtime)**: Some overlap—both `code/common/cmdlib.c` and this `q3asm/cmdlib.c` exist; the runtime engine reuses the same patterns but has its own copies to avoid linking offline tool code at runtime
- **Windows GUI layer (`_printf` broadcast)**: Integrates with "Q3Map Process Server" (implied to be a GUI tool monitoring build progress)

### Outgoing (what this file depends on)
- **Platform layer**: `sys/stat.h`, `sys/types.h`, `windows.h` (Windows), `unistd.h` (POSIX)
- **Standard C library**: `stdlib.h`, `string.h`, `time.h`, `stdarg.h`, `errno.h`
- **Compiler toolchain**: No engine-level dependencies; purely standalone utility code

## Design Patterns & Rationale

### 1. **Tool-Engine Path Duality**
The `qdir`/`gamedir`/`writedir` globals establish a **canonical directory convention** (`BASEDIRNAME = "quake"`). Tools must locate themselves within this hierarchy before operating:
```c
SetQdirFromPath(myargv[0]);  // or from a loaded file path
```
This enforces that all offline tools and the runtime engine share a single consistent filesystem layout. The `ExpandPath()` / `ExpandGamePath()` functions delegate absolute-path checking to this hierarchy, allowing tools to work relative to `qdir` without runtime environment variables.

**Rationale**: In a pre-web-era game engine with multiple game directories (id1, mods, mission packs) and distributed tooling, a mandatory path-rooted initialization ensures deterministic behavior across Windows, macOS, and Linux.

### 2. **Error Handling Polymorphism via Preprocessor**
Two implementations of `Error()` are compiled conditionally:
- `WIN_ERROR`: Posts a Windows `MessageBox` for GUI tools
- Default: Prints to console and exits (CLI tools, Unix)

**Rationale**: The tool ecosystem spans interactive (GUI-launched Radiant, Q3Map with server feedback) and batch (command-line BSPC, Q3ASM) scenarios. Posting a message box in a CLI environment would hang; printing to console in a spawned subprocess would be lost.

### 3. **Verbose Output Multiplexing**
`_printf()` combines:
- Standard `printf` to console (always visible to the invoker)
- Optional Windows message broadcast to a monitoring server (for GUI-integrated tools)

**Rationale**: GUI tools (Radiant, Q3Map GUI) spawn subprocesses (BSPC, map compilation, entity parsing). The subprocess output is captured via `GlobalAddAtom` + `PostMessage`, allowing the GUI to display progress/errors in real-time without console windows.

### 4. **Stateful Token Parser**
`COM_Parse()` maintains global state (`com_token`, `com_eof`), not a parser object:
```c
char com_token[1024];
qboolean com_eof;
```

**Rationale**: 
- Lightweight; no dynamic allocation for parser state
- Matches the era's convention (early 2000s C codebases favored static state over opaque handles)
- Used across config parsing, entity definitions, script processing where one file per tool invocation was typical

### 5. **Archive-on-Expansion Pattern**
`ExpandPathAndArchive()` couples path resolution with optional file copying to an archive directory:
```c
if (archive) {
    sprintf(archivename, "%s/%s", archivedir, path);
    QCopyFile(expanded, archivename);
}
```

**Rationale**: Build tools often need to track which source files were read during compilation for reproducibility or incremental builds. This integrates archival directly into the expansion layer.

## Data Flow Through This File

1. **Initialization Phase**
   - Tool calls `SetQdirFromPath(myargv[0])` or `SetQdirFromPath(loaded_file_path)`
   - Scans path backwards for `BASEDIRNAME` ("quake"), extracts `qdir` and `gamedir`
   - Establishes global path context for the tool session

2. **Argument/File Loading**
   - `ExpandWildcards()` (Windows only) expands `*.map` to file list
   - Tool iterates arguments, calls `ExpandPath()` or `ExpandGamePath()` to resolve relative→absolute
   - If archiving enabled, `ExpandPathAndArchive()` copies to archive dir

3. **Parsing Phase**
   - Tool reads config/entity/script file via `FS_ReadFile()` (engine) or direct `fopen()` (tools)
   - Repeatedly calls `COM_Parse()` to tokenize, checking `com_eof` for end-of-stream
   - Parses quoted strings, skips comments (`//`), handles whitespace

4. **Error Exit**
   - Tool logic detects error, calls `Error(fmt, ...)`
   - Platform-specific handler (GUI MessageBox or stderr) is invoked
   - Process exits with code 1

## Learning Notes

### What a developer studying this file learns:

1. **Pre-Modern Tool Architecture**: Offline tools in the Quake engine ecosystem were not modular plugins but standalone binaries sharing a common utility layer. Today, such tools would use a library (e.g., `libq3engine.so`) or be integrated into a monolithic editor. Here, `cmdlib.c` is copied/duplicated across tool projects (`q3asm/`, `q3map/`, `code/bspc/`) to avoid build dependencies.

2. **Platform Abstraction via Preprocessor, Not Vtables**: The engine era avoided C++ virtual tables for cross-platform code. Instead, `#ifdef WIN32` / `#ifdef __linux` branches within the same function provide platform-specific logic. This is efficient but less composable than modern trait-based abstractions.

3. **Global State as Design Norm**: Variables like `qdir`, `gamedir`, `com_token`, `verbose` are file-scoped globals, not parameters. This was common in C before dependency injection became standard. Initialization order (`SetQdirFromPath()` must run first) is implicit, not enforced by type.

4. **Stateful Tokenization**: `COM_Parse()` is a stateful stream parser that maintains position implicitly via pointer arithmetic. Modern parsers use iterators, AST nodes, or zero-copy views. This approach worked well for config files up to ~10KB but scales poorly.

5. **Filesystem as Configuration Store**: The assumption that tools can always locate themselves relative to `qdir` reflects an era where tools ran locally from disk. Modern systems with containerized builds, cloud storage, and asset pipelines would parameterize paths rather than hardcode search.

### Idiomatic to this engine / era, different in modern engines:

| Aspect | Quake III Era | Modern Engines |
|--------|---------------|---|
| **Path management** | Rooted at a fixed `BASEDIRNAME` ("quake") in the filesystem | Configuration files, environment variables, or CLI flags specify root(s) |
| **Verbosity control** | Global `verbose` flag | Logger interface with per-module levels (spdlog, zerolog, etc.) |
| **Error handling** | `Error()` calls `exit(1)` immediately | Exceptions, Result<T, E>, or error callbacks allowing cleanup |
| **Tool integration** | GUI broadcasts output via Windows messages to a monitor process | Stdout/stderr captured by IDE/build system, or structured logging to a service |
| **Tokenization** | Stateful stream parser with global token buffer | Lexer library (pest, nom, tree-sitter) or hand-written recursive descent |
| **Platform abstraction** | Preprocessor conditionals (`#ifdef WIN32`) | Abstraction traits or separate `platform/` modules |

## Potential Issues

1. **Buffer Overflows in `sprintf` / `strcat`**  
   Lines like `sprintf(filename, "%s%s", filebase, fileinfo.name)` and `strcat(temp, path)` use unbounded operations. If `filebase + fileinfo.name > 1024` or input path approaches 1024, overflow occurs. Modern code uses `snprintf()` with explicit limits.

2. **Wildcard Expansion Only on Windows**  
   `ExpandWildcards()` is stubbed on Unix/Linux (`void ExpandWildcards(...) {}`), so `*.map` arguments won't expand on those platforms. Shell expansion is relied upon instead, but this is inconsistent.

3. **Stateful Parser Not Thread-Safe**  
   Global `com_token`, `com_eof` mean `COM_Parse()` cannot be called concurrently. No issue in single-threaded tools, but problematic if tools ever parallelize parsing.

4. **Path Canonicalization Fragile**  
   The search for `BASEDIRNAME` in `SetQdirFromPath()` assumes the string exists in the path. If a tool is invoked from outside the `quake/` hierarchy, the error message is generic: `Error("SetQdirFromPath: no '%s' in %s", BASEDIRNAME, path)`. No fallback or environment-variable override.

5. **Silent Failure in `ExpandWildcards`**  
   If `_findfirst()` returns `-1` (no matches), the function silently returns without setting `*argc` to reflect the unmatched argument. The glob remains unexpanded, potentially confusing users.
