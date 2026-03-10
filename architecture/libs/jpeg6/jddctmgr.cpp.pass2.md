# libs/jpeg6/jddctmgr.cpp — Enhanced Analysis

## Architectural Role

This file is part of the **vendored IJG libjpeg-6 library**, bundled solely for runtime JPEG texture decompression in the renderer. It implements the IDCT (Inverse Discrete Cosine Transform) management layer—the infrastructure that selects and initializes the appropriate DCT algorithm variant and builds the quantization multiplier tables required for baseline JPEG decoding. Unlike the core engine subsystems, this file is **not architectural to Quake III itself**; it is a transparent dependency consumed only by `code/renderer/tr_image.c` (via `code/jpeg-6/jload.c`) during texture asset loading.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/jpeg-6/jload.c`** — The only runtime JPEG loader in the engine; calls `jinit_inverse_dct()` during decompression setup
- **`code/renderer/tr_image.c`** — Indirectly, when loading `.jpg` texture assets; the renderer delegates to libjpeg for decompression
- **Compile-time only:** `code/jpeg-6/j*.c` (other JPEG library modules) depend on declarations from `jdct.h` and `jpeglib.h`

### Outgoing (what this file depends on)
- **`jpeglib.h`, `jinclude.h`, `jdct.h`** — IJG libjpeg public and private headers; no external engine dependencies
- **Memory allocation:** Uses `cinfo->mem->alloc_small()` (JPEG's internal allocator, not qcommon Hunk)
- **Error handling:** Uses `ERREXIT`/`ERREXIT1` macros (JPEG's internal `setjmp`-based error scheme, not `Com_Error`)

---

## Design Patterns & Rationale

### Multi-Method IDCT Selection (Runtime + Compile-Time)

The file implements a **pluggable algorithm selection pattern**:

1. **Compile-time flags** (`DCT_ISLOW_SUPPORTED`, `DCT_IFAST_SUPPORTED`, `DCT_FLOAT_SUPPORTED`) determine which implementations are compiled in.
2. **Runtime cvar** (`cinfo->dct_method`) selects among available methods for full-size DCT blocks.
3. **Reduced-size blocks** (1×1, 2×2, 4×4) always use dedicated routines (`jpeg_idct_1x1`, etc.), falling back to `JDCT_ISLOW` tables.

**Rationale:** IDCT is performance-critical in JPEG decoding. Different CPU targets (x86 vs ARM, floating-point availability, SIMD capabilities) may favor different implementations. The JPEG standard leaves implementation flexible, so libjpeg provides three variants:
- **ISLOW**: Integer-only, highest precision, slowest
- **IFAST**: Integer with scaled coefficients, ~2× faster, slightly lower precision
- **FLOAT**: Floating-point, fastest on FPU-rich targets

### Quantization Table Preprocessing

Multiplier tables are **precomputed once per output pass**, not per coefficient:
- Raw quantization values from the JPEG bitstream are scaled by **AA&N (Arai, Agui, Nakajima) scale factors** to account for the DCT basis normalization
- For IFAST, coefficients are scaled by `aanscales` array (precomputed scaled by 14 bits) to avoid runtime division
- For FLOAT, true floating-point scaling is applied

This avoids recomputing scales thousands of times during the actual IDCT phase.

### State Caching via `cur_method[]`

The `my_idct_controller` struct caches which method each component's multiplier table is built for. If the method hasn't changed between passes, the table is **not rebuilt**—a micro-optimization for multi-scan JPEG files.

---

## Data Flow Through This File

```
Input (from jdinput.c / JPEG bitstream parsing):
  ├─ Quantization table pointer (JQUANT_TBL*)
  └─ Component configuration (DCT_scaled_size)
                    ↓
            start_pass() called at output pass init
                    ↓
          Select IDCT method based on:
            ├─ DCT_scaled_size (1, 2, 4, or DCTSIZE=8)
            ├─ cinfo->dct_method (if full-size)
            └─ Compile-time support flags
                    ↓
          Build multiplier_table from JQUANT_TBL:
            ├─ ISLOW: Direct copy of quantization values
            ├─ IFAST: Scale by aanscales[], downshift by (CONST_BITS - IFAST_SCALE_BITS)
            └─ FLOAT: Scale by aanscalefactor[] doubles
                    ↓
         Store method pointer in inverse_DCT[ci]
         Store table pointer in compptr->dct_table
                    ↓
      Output: Ready for jdcoefct.c to invoke the IDCT routine
             with precomputed multiplier table on every MCU block
```

---

## Learning Notes

### JPEG Decompression Architecture (IJG)
- **Separation of concerns:** jddctmgr handles *setup*, not *execution*. The actual IDCT (`jidctislow.c`, `jidctfst.c`, `jidctflt.c`) is a separate compilation unit.
- **Quantization fusion:** Modern JPEG decoders fuse dequantization into the IDCT for cache locality. IJG does this by precomputing multiplier tables rather than applying a separate dequantization step.
- **Scaling vs. precision trade-off:** IFAST sacrifices precision (16-bit fixed-point) for speed; ISLOW uses full 32-bit intermediate products. The choice is **not visible to the user**—output is always 8-bit samples.

### Idiomatic to This Era (Pre-2000s Software)
- **Conditional compilation** rather than runtime dispatch: Methods are compiled in or out at build-time (`#ifdef`), not selected via function pointers at runtime.
- **Macro-heavy:** `ERREXIT`, `SHIFT_TEMPS`, `DESCALE`, `MULTIPLY16V16` hide platform-specific optimizations and error handling.
- **No SIMD:** All variants are scalar. Modern JPEG decoders would use NEON/SSE, but that's absent here.
- **Manual memory pooling:** JPEG allocator uses a custom pool scheme (`JPOOL_IMAGE`), not libc malloc—gives finer control over lifetime and fragmentation.

### Contrast with Modern Engines
- Modern engines often use **hardware-accelerated texture decompression** (GPU support for BCn formats) or **fast runtime codecs** (ASTC, ETC2).
- Quake III bundles a CPU decompressor because it targets 1999–2001 hardware where JPEG was the practical choice for disk footprint.

---

## Potential Issues

### Limited Observability
- **No indication in the code** which method is actually selected. The `cur_method[]` cache has no logging; silent fallback to ISLOW if unsupported methods are requested.
- **If a developer adds a new IDCT variant** without updating this module, it will silently fail with `ERREXIT(cinfo, JERR_NOT_COMPILED)` rather than a clear error message.

### Floating-Point Precision (Minor)
- In `case JDCT_FLOAT`, the `aanscalefactor` array is hardcoded to 6 decimal places. Rounding errors accumulate; some JPEG test suites expect bit-for-bit identical output across platforms, which FP variants cannot guarantee.

### Coupling to JPEG Internals
- Any future change to the JPEG quantization table format or DCTSIZE would require updates here. The code assumes `DCTSIZE == 8` and `DCTSIZE2 == 64` (8×8 DCT blocks), which is universal in practice but not enforced by the JPEG standard for specialized profiles.
