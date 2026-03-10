# code/jpeg-6/jidctint.c — Enhanced Analysis

## Architectural Role

This file implements the slow-but-accurate integer IDCT variant used during JPEG texture decompression in the renderer's texture-loading pipeline. It is part of the vendored IJG libjpeg-6 library, which sits at the **image format support layer** — below `code/renderer/tr_image.c` and accessed through the lightweight JPEG loader stub at `code/jpeg-6/jload.c`. The file has no direct engine dependencies; all communication flows through libjpeg's own API. Rendering performance is indifferent to which IDCT variant is chosen (this one vs. faster approximations), since texture decompression is I/O-bound and happens at load time, not during frame rendering.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture loading pipeline** (`code/renderer/tr_image.c`):
  - Calls `R_LoadJPG()` (from `code/jpeg-6/jload.c`) during `R_FindImageFile()` / `R_CreateImage()` to load `.jpg` texture files.
  - Selector logic: only reached if the texture filename ends in `.jpg` and raw image data is not already cached.
  - Call frequency: once per unique JPG texture file during level load (texture cache hit avoids re-decompression).
- **libjpeg decompression control flow**:
  - `jpeg_idct_islow` is registered as the `IDCT` function pointer in `jpeg_decompress_struct` during decompressor setup.
  - Invoked once per 8×8 MCU block component in the image, in raster order, from `jddctmgr.c` (the "DCT manager" that coordinates all IDCT calls).
  - Typical JPEG texture: ~256×256 to ~1024×1024 pixels, yielding hundreds to thousands of MCU blocks per texture.

### Outgoing (what this file depends on)
- **No engine subsystems called directly.**
- **libjpeg internal macros and types** (from `jdct.h`, `jpeglib.h`, `jmorecfg.h`):
  - `DEQUANTIZE(coef, quantval)` — pre-computed quantization table lookup (supplied by `compptr->dct_table`).
  - `MULTIPLY(var, const)` — conditional macro: `MULTIPLY16C16` (16×16→32 multiply) for 8-bit samples, or full 32×32 multiply for 12-bit samples.
  - `DESCALE(value, bits)` — right-shift with rounding (used for fixed-point output scaling).
  - `IDCT_range_limit(cinfo)` — macro returning pointer to pre-computed clamping table (populated by `code/qcommon` / `code/renderer` shared init, not shown in this file).
- **No heap or filesystem I/O.**
- **No platform layer calls.**

## Design Patterns & Rationale

### 1. **Separated Passes (Column + Row IDCT)**
- The 2D IDCT is decomposed into two 1D passes: pass 1 processes columns, pass 2 processes rows.
- **Rationale**: Reduces code size and allows reuse of the same 1D kernel. Separability works because the DCT basis is a tensor product.
- **Tradeoff**: Two full passes + workspace buffer (256 bytes stack) vs. single direct 2D algorithm (would be more complex, no speed gain in software).

### 2. **Fixed-Point Arithmetic with Pre-Scaled Constants**
- All DCT basis coefficients are pre-multiplied by `2^CONST_BITS` (13) and stored as integer literals (`FIX_*` macros).
- Final output is descaled by shifting right `CONST_BITS + PASS1_BITS + 3 = 18` bits.
- **Rationale**: Avoids floating-point entirely (critical on CPUs without FPU; preserves reproducibility across platforms). The scaling factors are compile-time constants, so the compiler can fold them.
- **Tradeoff**: Careful management of scaling steps to avoid overflow or precision loss; the comments explain the bit-budget arithmetic.

### 3. **All-Zero AC Short-Circuit**
- Detects columns/rows where all AC terms are zero (common after quantization). Falls back to a single DCT-scaled DC copy.
- **Rationale**: Approximately 50% of columns and 5–10% of rows are all-zero in typical images. Skipping the full IDCT pipeline saves ~30 multiplies and ~20 adds per block.
- **Implementation**: Bitwise OR of all AC coefficient dequantized values; zero test before main algorithm.
- **Guarded Path**: Pass 2's zero-row test is conditional (`#ifndef NO_ZERO_ROW_TEST`) because the cost of the test may exceed the savings on very-high-multiplier hardware.

### 4. **Dequantization Fused into IDCT**
- Quantized coefficients are dequantized on-the-fly using the component's multiplier table (`compptr->dct_table`).
- No separate dequantize pass.
- **Rationale**: Reduces memory traffic (one read of quantized coefficient, immediately multiply, no temp write). Matches libjpeg's architecture where IDCT is the bottleneck.

### 5. **Range-Limiting via Pre-Computed Lookup Table**
- All output values are clamped into `JSAMPLE` (0–255) range using a pre-computed `sample_range_limit` table.
- **Rationale**: Avoids branching; lookup table is L1-cache-friendly. The table accounts for potential rounding artifacts that could produce out-of-range values before clamping.

## Data Flow Through This File

```
Input: 64 quantized DCT coefficients (JCOEFPTR coef_block)
       + component dequantization table (compptr->dct_table)

  ↓

  Pass 1: Column-wise 1D IDCT
    - Dequantize each column's 8 coefficients
    - Apply Loeffler IDCT algorithm with fixed-point arithmetic
    - Scale output up by 2^PASS1_BITS to preserve precision
    - Write 8×8 intermediate results to workspace buffer

  ↓

  Pass 2: Row-wise 1D IDCT
    - Read 8×8 rows from workspace
    - Repeat Loeffler IDCT algorithm
    - Descale by CONST_BITS + PASS1_BITS + 3 bits (total factor of 2^18)
    - Clamp to JSAMPLE range via range_limit table lookup
    - Write 8 rows × 8 samples directly into output_buf

Output: 64 pixel samples (8×8 MCU block) in output_buf[0..7][output_col..output_col+7]
        Each sample is a valid JSAMPLE (0–255 for 8-bit)
```

## Learning Notes

1. **Idiomatic Mid-1990s Fixed-Point Engineering**: This code exemplifies pre-FPU-era numerics: careful bit accounting, compile-time constant folding, and explicit precision preservation via scaling factors. Modern engines abstract this away (e.g., SIMD intrinsics or GPU compute shaders), but the mathematical structure remains identical.

2. **Loeffler–Ligtenberg–Moschytz Algorithm**: The choice of 12 multiplies and 32 adds (vs. the optimal 11 multiplies) reflects a tradeoff: the extra multiply is placed on a data path that can be pipelined independently, reducing critical-path latency. This is a **hardware-conscious algorithm design** — the paper title references "Practical Fast 1-D DCT Algorithms."

3. **Separability in Basis Decomposition**: The 2D IDCT = column pass + row pass is a consequence of the DCT basis being a separable tensor product. Same principle applies to 2D convolution, Fourier transforms, and many other signal-processing kernels. Understanding why separation works (vs. just memorizing the code) is key to recognizing where this pattern applies.

4. **Quantization as Lossy Compression**: The all-zero AC detection illustrates why JPEG is so effective at compression: quantization tables aggressively zero out high-frequency components that human vision is insensitive to. The decoder inherits this sparsity, allowing skip optimization.

5. **Vendor Code in Shipped Engines**: Quake 3 vendored libjpeg-6 in its source tree rather than linking a system library. This pattern (still common in 2005, less so today) ensures:
   - Reproducible builds (same libjpeg version everywhere)
   - No system dependency on a third-party library version
   - Ability to patch security issues in-tree without waiting for distro updates
   - Single reference implementation across platforms (Windows DLL, Linux .so, Mac framework)

6. **No Engine Coupling, Pure Math Leaf**: Unlike most game engine code, this file is **zero-dependency** on the engine core. It could be used in any application (camera firmware, medical imaging, etc.). This is the hallmark of a well-vendored library — it doesn't know or care that it lives inside Quake 3.

## Potential Issues

**None obvious from the code alone**, given the context:

- The `#if DCTSIZE != 8` syntax error is **intentional** (enforces compile-time assumption).
- Bit-budget arithmetic is carefully commented and matches IJG's published analysis.
- Range-limiting via table lookup is safe (the table size is guaranteed sufficient).
- No buffer overruns: workspace is stack-allocated (256 bytes), output array bounds are caller-supplied and trusted.
- The `#ifndef NO_ZERO_ROW_TEST` guard is a documented pragmatic trade-off, not a bug.

**Historical note**: On modern CPUs with out-of-order execution and SIMD, the penalty for the zero-row test may have inverted (i.e., disabling it could be faster). But that's a performance tuning opportunity, not a correctness issue.
