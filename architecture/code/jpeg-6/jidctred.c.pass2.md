# code/jpeg-6/jidctred.c — Enhanced Analysis

## Architectural Role

This file implements specialized inverse DCT routines for the **renderer's texture loading pipeline**. When JPEG textures are loaded via `code/renderer/tr_image.c` → `code/jpeg-6/jload.c` → libjpeg decompression, these functions are plugged into the libjpeg dispatch mechanism to decompress directly to reduced-size pixel buffers (4×4, 2×2, or 1×1), avoiding the cost of a full 8×8 IDCT followed by downsampling. This is critical for streaming and memory-efficient texture loading in the renderer's image cache.

## Key Cross-References

### Incoming (who depends on this file)
- **libjpeg decompression pipeline** (`jddctmgr.c`): Selects and registers `jpeg_idct_4x4`, `jpeg_idct_2x2`, or `jpeg_idct_1x1` as function pointers into the DCT manager dispatch table based on `DCT_scaled_size` at decompression startup
- **`code/renderer/tr_image.c`** (indirectly): Calls `R_LoadJPG` which invokes the full libjpeg decompression stack
- No direct engine calls to these functions; they are invoked entirely through the libjpeg callback mechanism

### Outgoing (what this file depends on)
- **`jinclude.h`, `jpeglib.h`, `jdct.h`**: All JPEG library internal headers; no engine subsystem dependencies
- **IDCT range-limit table** (`IDCT_range_limit` macro from `jdct.h`): Provided by the decompressor state; clamps output samples to valid range (`[0, MAXJSAMPLE]`)
- **Quantization tables** (via `dct_table` in `jpeg_component_info`): Set up by libjpeg's `jcdctmgr.c` during decompression init
- No outgoing calls to renderer, qcommon, or any other engine subsystem

## Design Patterns & Rationale

**Signal-processing decomposition**: The 2D reduced IDCT is decomposed into two 1D passes (columns, then rows), each replacing the standard 8-to-8 transform with an 8-to-N reduction. This is derived from the LL&M (Loeffler, Ligtenberg, Moschytz) algorithm used in `jidctint.c`, but simplified by discarding output samples that won't be used.

**Fixed-point acceleration**: All multiplications use pre-calculated `FIX_*` constants (lines 62–75) tailored to `CONST_BITS=13`. The macro `MULTIPLY` conditionally uses `MULTIPLY16C16` for 8-bit samples (faster) or full 32-bit multiplication for 12-bit samples, avoiding runtime FP cost. This is a classic embedded-graphics optimization pattern.

**Sparse-coefficient fast path**: Both passes check if all non-DC AC terms are zero (lines 169–178 in `jpeg_idct_4x4`, lines 270–279 in `jpeg_idct_2x2`). If true, output is trivially the scaled DC value repeated. This is critical: typical JPEG blocks are heavily quantized and have many zero coefficients, making the fast path dominant.

**Reduced-size specialization**: Rather than computing all 8 outputs and discarding, the algorithm computes only 4, 2, or 1 final values, eliminating ~50% of arithmetic. Column indices that won't contribute to the output (e.g., column 4 in 4×4 mode, lines 165–166) are skipped entirely.

**Double-buffering via workspace**: Intermediate column results are stored in a stack-allocated workspace, then read back in the row pass. This isolation allows each pass to operate independently and improves cache locality.

## Data Flow Through This File

1. **Input**: 64 quantized DCT coefficients (`coef_block[0..63]`), component's quantization table (`dct_table`), output buffer reference
2. **Pass 1 (columns)**:
   - For each column (skipping irrelevant ones), dequantize and apply reduced-size LL&M butterfly
   - Check sparse-coefficient fast path; if taken, write DC values to workspace
   - Otherwise, compute LL&M even/odd parts and write 4/2/1 scaled outputs to workspace rows
3. **Pass 2 (rows)**:
   - Read 4/2/1 workspace rows back as input columns
   - Repeat LL&M logic, producing final pixel samples
   - Clamp via `range_limit` lookup table and write directly to `output_buf[row][output_col..col+N]`

## Learning Notes

- **Era-appropriate optimization**: Fixed-point IDCT was standard in 1990s graphics engines. Modern engines use SSE/NEON or GPU shaders, but this demonstrates the careful hand-tuning that made real-time JPEG decompression feasible on early 2000s hardware.
- **Coefficient sparsity exploitation**: The zero-AC fast path reveals that JPEG quantization is *designed* to produce sparse blocks; the encoder and decoder collaborate on this assumption.
- **Reduced-size as first-class operation**: Rather than treating 4×4/2×2 output as a downsampling step, this file treats it as a distinct transform mode. Modern texture streaming (e.g., mipmap generation) sometimes uses analogous approaches.
- **Idiomatic JPEG library design**: The use of dispatch tables, macro-based polymorphism (`MULTIPLY`, `DEQUANTIZE`, `DESCALE`), and `JSAMPLE` type abstraction (8 or 12 bits) shows the JPEG library's focus on portability and customization without generics.

## Potential Issues

- **Conditional compilation guard** (`#ifdef IDCT_SCALING_SUPPORTED`): If the engine's `jdct.h` doesn't define this, these routines are never compiled or used. Dead code if the feature is disabled.
- **Stack allocation of workspace**: `jpeg_idct_4x4` allocates `int workspace[DCTSIZE*4]` (32 ints = 128 bytes on 32-bit systems), and each column loop iteration writes to it. On deeply nested call stacks or embedded systems with limited stack, this could be risky. (Not inferable as a practical issue here, but worth noting for ports.)
- **Hardcoded FIX constants assume CONST_BITS=13**: Lines 62–75 are pre-calculated for this specific configuration. If `jdct.h` changes `CONST_BITS`, these constants become incorrect, causing silent numerical errors in decompressed pixels. The `#else` branch (lines 77–89) falls back to runtime `FIX()` macros, which are slower but safe.
- **No error checking**: Invalid or corrupted DCT coefficient blocks will not be detected; they'll simply produce garbage pixel values. Rely on upstream libjpeg validation.
