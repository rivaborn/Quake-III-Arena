# code/jpeg-6/jdtrans.c — Enhanced Analysis

## Architectural Role

This file is part of the vendored IJG libjpeg-6 library (code/jpeg-6/), a pure utility for texture asset loading with zero integration into the core Q3A engine architecture. It provides the transcoding decompression pathway—reading raw DCT coefficients without full image reconstruction—used when the renderer loads JPEG textures. Unlike the rich subsystems documented in the architecture (qcommon, renderer, server, game), this is external-library code that bridges Q3A's asset pipeline with the standard IJG public API.

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c** — sole consumer; calls `jpeg_read_coefficients` during texture asset load (though exact callsite not visible in provided context, this is the typical IJG usage pattern)
- **Public API callers** — any application that transcodes JPEG files (e.g., tools); not called internally by server/game/client

### Outgoing (what this file depends on)
- **Internal JPEG modules** (`jdhuff.c`, `jdphuff.c`, `jdcoefct.c`) — entropy decoder and coefficient buffer initialization functions
- **JPEG memory manager** (`cinfo->mem->realize_virt_arrays`) — virtual array materialization
- **JPEG input controller** (`cinfo->inputctl->consume_input`, `start_input_pass`) — stream feeding
- **JPEG error handler** (`cinfo->err->error_exit` via `ERREXIT` macros)
- **Zero external dependencies** — no qcommon, renderer, or platform layer calls

## Design Patterns & Rationale

**State Machine**: The decompressor state (`DSTATE_READY` → `DSTATE_RDCOEFS` → `DSTATE_STOPPING`) ensures one-time initialization and idempotency on re-entry. Re-entry skips re-initialization; this supports suspending data sources.

**Deferred Module Initialization**: `transdecode_master_selection` instantiates only the minimal subsystem subset needed (entropy decoding + coefficient buffering), **omitting** the full IDCT, color conversion, upsampling, and quantization chains that a `jdmaster.c` full decompressor would set up. This reflects the transcoding use case: lossless coefficient extraction without reconstruction.

**Virtual Array Abstraction**: Coefficient arrays are not materialized until `realize_virt_arrays` is called, deferring allocation to the memory manager. This is a memory-efficiency pattern for large images where the full decoded image may not fit in contiguous RAM.

**Progress Estimation Heuristic**: Scan count is estimated (2 DC + 3×N AC for progressive; N for multi-scan; 1 for single-scan) to guide progress monitors. The ratcheting mechanism (`pass_limit += total_iMCU_rows`) adapts when the file has more scans than initially guessed, keeping the progress bar responsive.

## Data Flow Through This File

1. **Input**: JPEG stream via `cinfo->inputctl->consume_input`
2. **Entropy Decoding**: Raw bitstream → MCU-organized DCT coefficient blocks (Huffman or progressive)
3. **Buffering**: Coefficients → virtual arrays (`coef_arrays`)
4. **Output**: Pointer to `jvirt_barray_ptr` array (one per color component)
5. **Feedback**: Progress monitor hook called each iteration for UI responsiveness
6. **Return Value**: `NULL` on suspension (data source not ready); otherwise array pointers

The loop runs until `JPEG_REACHED_EOI`, making this a **blocking, whole-file operation** from the caller's perspective.

## Learning Notes

**Transcoding Lesson**: Reading coefficients without decoding them (no IDCT) is the path to lossless re-encoding. Contrast this with full decompression (tr_image.c → renderer) where textures undergo full reconstruction and color space conversion.

**Modular Initialization**: The split between `jpeg_read_coefficients` (public entry, loop management) and `transdecode_master_selection` (module setup) is clean separation of concerns. Modern engines (ECS-based) use similar patterns for subsystem boot-up.

**IJG Era Convention**: The `JPP` macro syntax, state constants (`DSTATE_*`), and error exit style (`ERREXIT`) reflect late-1990s C library conventions. Macros like `LOCAL`/`GLOBAL` are portability shims absent from C99/C11 codebases.

## Potential Issues

- **Arithmetic Coding Hard Fail** (line 85): `JERR_ARITH_NOTIMPL` aborts transcoding. If a JPEG uses arithmetic coding (rare but valid), decompression fails immediately rather than gracefully downgrading or skipping. Not a realistic problem for Q3A assets but worth noting for asset authoring.
- **Progressive Compile-Time Gate** (line 87): If `D_PROGRESSIVE_SUPPORTED` is not defined, progressive JPEGs cause `JERR_NOT_COMPILED`. This is a build-time footprint tradeoff (unlikely to affect shipped Q3A, which likely enables it).
- **Progress Pointer Dereference** (line 103): `cinfo->progress` is checked for NULL before use, but the assumption that it remains valid across the entire loop is not asserted. Defensive coding would re-check or document the invariant.
- **Scan Count Underestimation Logic**: The ratcheting assumes `total_iMCU_rows` remains stable; if the BSP/asset system alters image metadata during processing, the estimate could diverge.
