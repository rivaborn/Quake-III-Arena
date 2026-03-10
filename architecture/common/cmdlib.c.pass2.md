# common/cmdlib.c — Enhanced Analysis

## Architectural Role

`cmdlib.c` is the **foundational utility library for Quake III's offline toolchain**, not the runtime engine. It serves as the shared base for three independent tool processes—**q3map** (BSP compiler), **bspc** (AAS navigation compiler), and **q3radiant** (level editor)—providing filesystem abstractions, path resolution, and string tokenization that all three tools rely on at startup and throughout execution. Unlike `qcommon/` (which is runtime-exclusive and handles virtual filesystems, networking, and VMs), `cmdlib` is a simple, tool-specific foundation with no dependencies on rendering, collision, or game logic.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map** — calls `SetQdirFromPath` to establish working paths, uses `LoadFile`/`SaveFile` for BSP I/O, `COM_Parse` for shader/BSP parsing
- **bspc** — calls `SetQdirFromPath` for AAS setup, uses `LoadFile`/`CopyFile` for binary AAS serialization
- **q3radiant** — calls `SetQdirFromPath` on startup, uses `LoadFile`, `SaveFile`, path expansion for texture/model loading
- **No runtime engine dependencies** — `code/qcommon`, `code/client`, `code/server` do **not** link or call cmdlib; they use their own `qcommon/files.c` and error handling

### Outgoing (what this file depends on)
- **Platform layer** only: `sys/stat.h`, `sys/types.h`, `time.h`, OS-specific `<windows.h>`, `<direct.h>`, `<io.h>` (Windows), `getwd`/`mkdir` (POSIX)
- **cmdlib.h** header (declares all symbols)
- **No qcommon dependencies** — stands alone; no calls to `Com_*`, `FS_*`, or VM infrastructure

## Design Patterns & Rationale

**Tool-time vs. runtime separation.** Quake III cleanly partitions the codebase: `code/` is the runtime engine (client, server, renderer, game logic); `common/` and the tool directories are **build-time only**. `cmdlib` exploits this boundary by:

- Using **global path state** (`qdir`, `gamedir`, `writedir`) set once at startup — acceptable for single-threaded tool processes, but would be dangerous in multi-threaded engine
- **Simple file I/O** (`fopen`/`fread`/`fwrite`) instead of the runtime's virtual filesystem (`FS_*` layer)
- **C `Error()` macro for fatal exits** instead of runtime's `Com_Error` longjmp mechanism
- **Synchronous I/O** with no async streaming or buffering — appropriate for offline tools that need deterministic execution

**Token parsing design.** `COM_Parse` is a lightweight, **forward-only** parser designed for tool-time scripts (shader definitions, BSP entity strings, config files). It has no `ungetc` or lookahead — a conscious choice for line-oriented parsing where single-pass suffices. This contrasts with runtime config/console parsing, which may want more sophistication.

## Data Flow Through This File

**Startup phase (all three tools):**
1. Tool entry point calls `SetQdirFromPath(argv[0])` to locate the engine root (`"quake"` directory)
2. Globals `qdir`, `gamedir`, `writedir` are populated; subsequent `Expand*` calls resolve relative paths
3. Archives are optionally configured via `archive` / `archivedir` globals

**Processing phase:**
1. Tools call `LoadFile(path)` → entire file into heap buffer (null-terminated)
2. Call `COM_Parse()` in a loop to tokenize config/script streams
3. Call `SaveFile()` or `QCopyFile()` to persist compiled output
4. `_printf` (with Win32 broadcast) provides progress feedback; `Error()` halts on failure

**Byte-order conversions** are applied transparently when reading/writing binary formats (e.g., AAS files, BSP headers).

## Learning Notes

**Engine architecture lesson:** Modern game engines (Unity, Unreal, Godot) integrate tools into the same process; Quake III separates them completely. The trade-off: **each tool is a standalone executable** (no shared state, simpler debugging) but **code duplication** (math, string utilities duplicated in `common/` and `code/q_shared.c`). This was pragmatic for late-1990s development.

**Idiomatic patterns from the era:**
- No memory pooling or allocator plugins — tools use `malloc` directly
- No logging framework — `_printf` is a simple global with platform-specific multiplexing (Win32 GUI window broadcast via `PostMessage`)
- Path strings stored in fixed-size buffers (1024 bytes) — stack-allocated, no heap indirection
- CRC-16 computed as a side effect of file reads (pre-computed lookup table `crctable`) — a micro-optimization no longer necessary

**Contrast with qcommon:**
| Aspect | cmdlib (tools) | qcommon (runtime) |
|--------|---|---|
| Filesystem | Raw `fopen`/POSIX | Virtual FS with `.pk3` merging |
| Memory | Direct `malloc` | Hunk allocator, zone memory |
| Error handling | `Error()` → exit(1) | `Com_Error` → longjmp + shutdown |
| Byte-order | Explicit `LittleShort` calls | Applied at serialize boundary |
| Threading | Synchronous (single-threaded tools) | Multi-threaded safe (optional SMP renderer) |

## Potential Issues

1. **Buffer overflow in `COM_Parse`:** No bounds checking on `com_token[1024]`. A pathological token (e.g., an unquoted string of 2000 characters) will overflow.

2. **Path manipulation with fixed buffers:** `qdir`, `gamedir`, `writedir`, `filename` all use 1024-byte stacks. Deeply nested paths or long mod names can overflow.

3. **Deprecated POSIX functions:** `getwd()` is obsolete (replaced by `getcwd()` in modern POSIX). `Q_getwd` on non-Windows uses the unsafe version.

4. **No bounds on string operations:** `strcpy`, `strcat`, `sprintf` used throughout without length arguments. Caller must ensure strings fit.

5. **Platform-specific divergence:** Win32 wildcard expansion (`ExpandWildcards`) is a no-op on POSIX. Tools behave differently on different platforms when called with glob patterns.

6. **IPC via undocumented Win32 window messages:** `_printf` broadcasts to a `"Q3Map Process Server"` window (if found) via `RegisterWindowMessage` + `GlobalAddAtom`. This is fragile (window may not exist, message name collisions) and not documented in the source.
