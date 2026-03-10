# libs/jpeg6/jpegint.h — Enhanced Analysis

## Architectural Role

This file is the **internal architecture contract** for the vendored IJG libjpeg-6 library used exclusively by the renderer's texture loading pipeline. It defines the modular vtable-based interface through which the JPEG library's compression and decompression subsystems communicate, allowing the renderer (`tr_image.c` via `jload.c`) to decompress JPEG texture data into raw RGB samples. The file itself is not engine-specific; it's a faithful preservation of the original IJG library structure, layered into the Quake III codebase as a static dependency for offline texture asset loading.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** (indirectly via `code/jpeg-6/jload.c`): Calls JPEG decompression functions to load `.jpg` texture assets during renderer initialization and runtime texture cache misses
- **All `code/jpeg-6/*.c` modules**: Every JPEG implementation file includes `jpegint.h` to access the internal module vtables and shared utilities
- No other subsystems import this header; collision, network, game, and bot code have zero dependency on JPEG infrastructure

### Outgoing (what this file depends on)
- **No external subsystem dependencies**: This is pure library specification—no calls into engine systems, cvar, memory managers, or collision code
- **Platform layer** (implicit): Individual JPEG modules may call `malloc`/`free` or equivalent through `jmemmgr.c`, but those are hidden from this header
- **Renderer** consumes only the public API (`jpeglib.h`), never this internal header directly

## Design Patterns & Rationale

**Vtable-based module architecture**: Each JPEG subsystem (e.g., `jpeg_entropy_encoder`, `jpeg_downsampler`, `jpeg_color_converter`) is a struct of function pointers initialized at runtime. This allows:
- **Swappable implementations**: Different JPEG profiles (baseline, progressive, lossless) can plug in different modules without recompilation
- **Separation of concerns**: Encoder/decoder/preprocessor code is cleanly layered; reachability is explicit through vtable chaining
- **Memory efficiency**: Modules only allocated when needed for the specific codec mode

**Macro-based infrastructure**:
- `JMETHOD()` macro (likely defined in a parent header) abstracts platform-specific function pointer syntax
- `JPP()` macro for function declarations likely handles ANSI vs. K&R C compatibility
- `JOCTET`, `JSAMPLE`, `JBLOCKROW` etc. are type abstractions supporting 8-bit, 12-bit, and 16-bit JPEG variants without recompilation

**Buffer mode enumeration** (`J_BUF_MODE`): Encodes the state machine for multi-pass operations (e.g., two-pass color quantization), allowing progressive buffering strategies without per-client specialization.

**State machine constants** (`CSTATE_*`, `DSTATE_*`): Hard-coded state values (100–210 range) enforce a strict ordering for state validation checks in `jdapi.c`.

## Data Flow Through This File

```
Engine Initialization
  └─> Renderer init (tr_init.c)
       └─> jload.c calls jpeg_decompress()
            └─> libjpeg internal vtables (this header)
                 └─> Module initialization (jinit_*) → Process pipeline
                      ├─> Marker reader (SOI, APP, etc.)
                      ├─> Entropy decoder
                      ├─> Inverse DCT + dequantization
                      ├─> Upsampler
                      └─> Color deconverter
                           └─> Raw RGB8 samples
                                └─> tr_image.c (texture upload)
```

**State transitions**: Decompressor lifecycle follows `DSTATE_START → DSTATE_INHEADER → DSTATE_READY → DSTATE_SCANNING → DSTATE_STOPPING`, enforced by module-state checks calling `setup_*` and `finish_*` methods.

**Memory layout**: Virtual arrays (referenced by `jvirt_sarray_control`, `jvirt_barray_control` stubs) abstract the difference between single-pass (streaming) and multi-pass (buffered) coefficient storage, critical for resource-constrained game engines.

## Learning Notes

**Late-1990s C library design**: This header exemplifies pre-STL, pre-C99 portable C practices:
- **No dynamic dispatch abstractions**: Vtables are manually constructed in per-module `.c` files
- **Linker compatibility shims**: The `NEED_SHORT_EXTERNAL_NAMES` block demonstrates support for brain-damaged 1980s linkers with 6-character symbol limits (e.g., Digital Equipment Corp. machines)
- **Platform-specific macros** (`RIGHT_SHIFT_IS_UNSIGNED`): Encodes known compiler quirks (unsigned vs. signed right-shift behavior varies by architecture)

**Modern contrast**: Contemporary engines (Unity, Unreal, Godot) use pluggable codec systems (FFmpeg, libavif) with versioned ABI contracts, whereas JPEG-6's design assumes a single monolithic library recompiled per platform.

**Architectural lessons**:
- The two-pass quantizer pattern (dummy pass for palette computation, then quantized output) influenced modern deferred-rendering and pre-pass architectures
- The separation of "coefficient buffer control" from "upsampler" shows intentional layering—decoupling color/sampling concerns from frequency-domain arithmetic

## Potential Issues

- **No error codes defined**: Modules use implicit error semantics (bool return, side-effect logging). Modern code would benefit from explicit error enums
- **Incomplete type stubs** (lines 365–370): The conditional `#ifdef INCOMPLETE_TYPES_BROKEN` forward declarations are a workaround for old C compilers unable to track opaque struct types, increasing compile-time coupling
- **Hard-coded array sizes**: `process_APPn[16]` assumes max 16 application markers; non-standard markers beyond `APP0`–`APPF` will be silently ignored
- **No thread-safety annotations**: If the renderer ever multi-threads JPEG decompression per texture, race conditions on global state (e.g., `aasworld` analog for JPEG) would go undetected
