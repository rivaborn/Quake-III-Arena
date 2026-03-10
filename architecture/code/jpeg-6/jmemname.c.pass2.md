# code/jpeg-6/jmemname.c — Enhanced Analysis

## Architectural Role
This file implements the **pluggable memory manager** abstraction for the vendored IJG libjpeg-6 library. It's invoked exclusively during texture asset loading (via `tr_image.c` → `jload.c` → libjpeg internals), never in the hot frame loop. The file demonstrates the JPEG library's core design philosophy: portability via conditional compilation and swappable subsystem implementations (`jmemname.c` vs. `jmemansi.c` vs. `jmemnobs.c` for different platform constraints).

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c**: Loads JPEG textures during asset initialization; transitively calls this module's functions through libjpeg's public `jpeg_create_decompress` → internal `(*memory_manager)()`
- **Entire libjpeg-6 library**: `jmemname.c` is one of three mutually-exclusive memory-manager implementations; selected at compile time via Makefile
- **Indirect from engine subsystems**: Texture loading occurs during map load and shader parsing, but texture assets are cached; this module is **not re-invoked per frame**

### Outgoing (what this file depends on)
- **Platform libc**: `malloc`, `free`, `fopen`, `fclose`, `fseek`, `unlink`/`remove`, `mktemp` (or manual polling)
- **System errno**: `ENOENT` constant (only if `NO_MKTEMP` is set)
- **JPEG library internals** (`jpeglib.h`, `jmemsys.h`, `jinclude.h`): Callback function signatures, error/logging macros (`ERREXIT`, `TRACEMSS`), port-specific macro definitions

## Design Patterns & Rationale

### 1. **Pluggable Subsystem Pattern**
IJG JPEG provides three memory-manager implementations, selectable at compile time:
- **`jmemname.c`** (this file): Explicit temp file naming (requires user configuration)
- **`jmemansi.c`**: ANSI-conformant with `tmpfile()` and full memory-limit querying
- **`jmemnobs.c`**: No backing store (pure in-memory, for systems with ample RAM)

This design allowed the library to target 1990s embedded systems, DOS, Unix servers, and mainframes simultaneously. The engine chooses `jmemname.c`, implying the build targets systems that lack ANSI compliance or `tmpfile()` support (typical of older Unix derivatives).

### 2. **Conditional Compilation for Portability**
```c
#ifdef NO_MKTEMP           // Two temp-file generation strategies
#ifdef NO_ERRNO_H          // errno may not exist on ancient systems
#ifdef DONT_USE_B_MODE     // Text-mode vs. binary-mode file I/O (Win32 vs. Unix)
#define TEMP_DIRECTORY     // Configurable temp location (Unix: /usr/tmp, could be TEMP env var)
```
This reflects 1990s portability constraints: no standard `tmpfile()`, no guaranteed `errno.h`, Windows text-mode files corrupt binary data. Modern engines would simply use `mkstemp()` and native temp directories.

### 3. **Method Pointer Installation (IoC-like)**
```c
jpeg_open_backing_store() {
  info->read_backing_store = read_backing_store;
  info->write_backing_store = write_backing_store;
  info->close_backing_store = close_backing_store;
}
```
The JPEG library uses function pointers in `backing_store_info` to dispatch I/O. This allows hot-swapping implementations without virtual method tables (which were expensive in 1992). Modern code would use vtables or strategy objects.

### 4. **Idiomatic: No-Op Distinction Between "Small" and "Large" Allocations**
```c
jpeg_get_small() { return malloc(sizeofobject); }
jpeg_get_large() { return malloc(sizeofobject); }  // identical implementation
```
The API exposes a distinction (small vs. large) for systems with segmented memory (80x86 `NEAR` vs. `FAR` pointers). This engine targets flat-memory architectures, so both are identical wrappers. The `FAR` keyword is vestigial and ignored by modern compilers.

## Data Flow Through This File

**Texture Loading Path (one-time, not per-frame):**
1. Engine calls `R_LoadImage()` for a JPEG texture
2. Libjpeg calls `jpeg_create_decompress()` → `jpeg_mem_init()` (reset file counter, return 1 MB limit)
3. JPEG library allocates internal buffers via `jpeg_get_small`/`jpeg_get_large` (malloc)
4. If intermediate image buffers exceed 1 MB, library calls `jpeg_open_backing_store()`
   - `select_file_name()` generates unique `/usr/tmp/JPG*.TMP` path (or polls filesystem if no `mktemp`)
   - `fopen(RW_BINARY)` creates temp file; function pointers installed into `backing_store_info`
5. JPEG decoding proceeds; intermediate scanlines spilled via `write_backing_store()` (fseek + fwrite) if needed
6. `read_backing_store()` fetches back spilled data during final decode pass (fseek + fread)
7. `close_backing_store()` deletes temp file (`unlink`)
8. `jpeg_mem_term()` (no-op cleanup)
9. Decoded image uploaded to GPU; temporary allocations freed via `jpeg_free_small`/`jpeg_free_large` (free)

**State Lifecycle:**
- `next_file_num`: Incremented per backing-store object, allowing multiple simultaneous JPEG decodes (rare in engine, but safe)
- Temp file names are **not cleaned up by this module**; caller must invoke `close_backing_store` at end of decode

## Learning Notes

### What's Idiomatic to This Engine/Era
1. **Vendor Library Isolation**: libjpeg is treated as an opaque black box; no assumptions about internal structure
2. **Compile-Time Configuration Over Runtime Detection**: No attempt to query OS for available memory (1 MB hardcoded default)
3. **Zero Abstraction for Flat Memory**: The `FAR` keyword and small/large distinction are noise on modern hardware
4. **Manual Temp File Management**: Pre-`mkstemp()` era; `mktemp()` is inherently unsafe (race condition between name generation and creation), but was acceptable for batch asset loading
5. **No Virtual Method Tables**: Method pointers in a struct, not vtable indirection; typical of embedded C code

### Modern Equivalents
- **Memory manager pluggability**: Replaced by dependency injection or static polymorphism (C++ templates)
- **Compile-time feature flags**: Now use `#[cfg(...)]` (Rust), `Optional<T>` (Java), or feature flags in package managers
- **Temporary file creation**: Use `mkstemp()` (POSIX) or `GetTempFileName()` (Win32 API), not `mktemp()`
- **Memory limit querying**: Call `sysconf(_SC_PHYS_PAGES)` (Unix) or `GlobalMemoryStatus()` (Windows); or trust the OS to kill the process via OOM killer
- **File I/O patterns**: Use buffered/memory-mapped I/O; avoid random seeks to disk for intermediate encoding state

### Game Engine Concepts
This file sits outside typical engine architecture patterns:
- **Not ECS**: No entity/component abstraction; pure functional memory management
- **Not a system**: No per-frame update or per-entity simulation
- **Not a subsystem**: Just a thin wrapper around malloc + file I/O, with no state that persists across frames
- **Closer to:** Utility library or abstraction layer (like how modern engines use `jemalloc` or `mimalloc` for memory pooling)

The JPEG library's existence in the renderer pipeline reflects a pre-GPU-texture-compression era. Modern game engines decompress JPEG at tool time and ship BC/ETC2 formats; libjpeg is retained here only for backward compatibility with loosely-defined `.shader` file workflows.

## Potential Issues

1. **Race condition in `NO_MKTEMP` path**: The loop checks `fopen()` → close → retry. Between the close and `next_file_num++`, another process could create the same file, causing a collision. Should use `O_EXCL` flag or `mkstemp()` instead.

2. **Memory availability query is a compile-time lie**: `jpeg_mem_available()` returns `cinfo->mem->max_memory_to_use - already_allocated`, which will never match actual free RAM. Modern engines should call `sysinfo()` at runtime or trust the OS to OOM-kill.

3. **Vestigial code complexity**: The `FAR` keyword, small/large distinction, and compile-time configuration (`NO_MKTEMP`, `NO_ERRNO_H`) add maintenance burden for zero modern benefit. Could be eliminated if targeting only flat-memory POSIX or Win32.

4. **No cleanup guarantee on crash**: If the process dies during texture load, temp files in `/usr/tmp/` are orphaned. Modern implementations should register a signal handler (`NEED_SIGNAL_CATCHER` comment hints at this).
