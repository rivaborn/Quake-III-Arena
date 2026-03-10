# code/jpeg-6/jcinit.c — Enhanced Analysis

## Architectural Role
This file orchestrates JPEG *compression* initialization for the vendored libjpeg-6 library. Unlike the renderer's texture loading (which uses decompression via `jload.c`), `jcinit.c` supports tools, the level editor, or offline image processing that need to *encode* JPEG. It is entirely decoupled from the game runtime—included in the build but not executed during gameplay.

## Key Cross-References
### Incoming (who depends on this file)
- Entry point: `jpeg_start_compress()` in `jcapistd.c` → calls `jinit_compress_master()`
- Build consumers: Likely `q3radiant/` (level editor) or build tools if they export screenshots/images to JPEG
- **Not called by:** renderer, game VM, botlib, or server at runtime

### Outgoing (what this file depends on)
- Calls subsystem `jinit_*` functions defined across:
  - `jcmaster.c` — parameter validation
  - `jccolor.c` — color conversion
  - `jcsample.c` — chroma downsampling
  - `jcprepct.c` — preprocessing controller
  - `jcdctmgr.c` — forward DCT
  - `jchuff.c` / `jcphuff.c` — Huffman encoders (sequential/progressive)
  - `jccoefct.c` / `jcmainct.c` — coefficient/main controllers
  - `jcmarker.c` — JFIF/JPEG marker output
- Reads: `cinfo->mem` (memory manager), `cinfo->dest` (output sink), parameter flags

## Design Patterns & Rationale
- **Modular subsystem initialization**: Each encoder component has a dedicated `jinit_*` function; the orchestrator wires them in correct dependency order. This is idiomatic to pre-1990s C library design, enabling static linking of optional features.
- **Feature gating via compile-time and runtime flags**: `C_PROGRESSIVE_SUPPORTED` controls progressive Huffman linking; `cinfo->arith_code` and `cinfo->progressive_mode` branch at runtime. Arithmetic coding is intentionally stubbed (`ERREXIT`), likely for patent/license reasons.
- **Deferred memory allocation**: Virtual arrays are allocated only after all subsystems declare their needs via `realize_virt_arrays()`. This batch-allocation pattern minimizes fragmentation in the hunk allocator.
- **Exception handling via macros**: `ERREXIT(cinfo, error_code)` implements a `longjmp`-based error handler, requiring careful cleanup discipline from init functions.

## Data Flow Through This File
**Input**: Caller provides a fully-populated `j_compress_ptr cinfo` with:
- Image dimensions, color space, subsampling factors
- Compression quality, progressive/sequential/arithmetic mode
- Output destination (`cinfo->dest` file writer)

**Transformation**:
1. Validate parameters and allocate master state
2. Conditionally initialize preprocessing (color → chroma-reduced YCbCr) or skip for raw data
3. Initialize forward DCT module
4. Select entropy encoder (Huffman sequential/progressive, or error)
5. Initialize buffering controllers (full-image buffer only if multi-pass)
6. Wire marker writer
7. Batch-allocate virtual arrays
8. Write SOI (Start of Image) marker immediately

**Output**: All `cinfo->*` subobject pointers populated; data-path ready to accept scanlines; first JPEG marker emitted.

## Learning Notes
- Demonstrates the "manual factory" pattern common in C libraries pre-OOP. Modern C++ engines would use virtual constructors or builder patterns.
- The conditional preprocessing chain shows how video codecs adapt to input format (raw YCbCr vs. RGB requiring conversion).
- Virtual array deferral is elegant for memory-constrained systems: allows subsystems to declare needs without early commitment.
- Arithmetic coding stub demonstrates how patent-burdened features were historically disabled; modern JPEG uses it more freely.
- The split between `jcapistd.c` (call to `jinit_compress_master()`) and this file is intentional—allows transcoding libraries to link only the decompressor, not the full encoder.

## Potential Issues
- **Compile-time feature loss**: If `C_PROGRESSIVE_SUPPORTED` is undefined, progressive JPEG encoding silently fails at runtime with `JERR_NOT_COMPILED`. No graceful fallback to sequential.
- **Arithmetic codec hardcoded**: Any attempt to enable arithmetic coding triggers immediate `ERREXIT`, making it impossible to add support without code changes.
- **No subsystem state validation**: Each `jinit_*` function is trusted to succeed; no checks for partial initialization if an early call fails.
