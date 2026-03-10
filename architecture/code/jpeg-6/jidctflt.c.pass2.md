# code/jpeg-6/jidctflt.c — Enhanced Analysis

## Architectural Role
This file is a **leaf node in the renderer's texture asset pipeline**. It implements the floating-point IDCT variant used by the vendored IJG libjpeg-6 library during JPEG decompression. Textures are loaded by `code/renderer/tr_image.c` (likely via `jload.c`), and any JPEG textures are decompressed per 8×8 block through this function, producing pixel-domain samples fed directly to GPU upload or further processing. The separable 2-pass algorithm and sparsity optimizations minimize CPU time during decompress, which happens once per texture at load time (not per-frame critical path).

## Key Cross-References

### Incoming (who depends on this)
- **`code/jpeg-6/jddctmgr.c`** — The DCT manager module that dispatches IDCT operations. It invokes `jpeg_idct_float()` (or alternative IDCT variants) once per MCU block during progressive/sequential JPEG decompression.
- **`code/renderer/tr_image.c`** — Indirectly, via `code/jpeg-6/jload.c`. The renderer's image loader calls the JPEG library's public decompression API, which internally routes to this function.
- **`code/jpeg-6/jmorecfg.h`** — Defines the conditional `#define IDCT_FLOAT_SUPPORTED` that gates this entire file's compilation.

### Outgoing (what it depends on)
- **`code/jpeg-6/jdct.h`** — Provides private DCT subsystem declarations: `IDCT_range_limit()` macro, `DESCALE()` macro, `RANGE_MASK`, `FAST_FLOAT` type alias, `FLOAT_MULT_TYPE`.
- **`code/jpeg-6/jpeglib.h` / `jinclude.h`** — Core IJG types: `j_decompress_ptr`, `jpeg_component_info`, `JCOEF`, `JSAMPLE`, `JDIMENSION`, `JSAMPARRAY`, `JSAMPROW`.
- **`cinfo->sample_range_limit`** — Populated externally by `code/jpeg-6/jdmaster.c` (`prepare_range_limit_table()`). A 384-entry lookup table mapping signed indices into valid `JSAMPLE` (0–255) range.
- **`compptr->dct_table`** — The dequantization multiplier table for the component, pre-computed by `jddctmgr.c` during decompressor setup. Type is `FLOAT_MULT_TYPE *` (likely `FLOAT` or `FAST_FLOAT`).

## Design Patterns & Rationale

### 1. **Floating-Point Accuracy Trade-off**
The file uses `FAST_FLOAT` (typically `float`) throughout, not fixed-point integer arithmetic. This is intentional:
- **Why:** JPEG quantization already destroys precision; floating-point reconstruction avoids compounding that loss via rounding artifacts.
- **Trade-off:** Slower than integer IDCT variants but more accurate on all platforms (integer variants may differ due to overflow/truncation behavior).
- **Platform conditional:** Compilation depends on `DCT_FLOAT_SUPPORTED`; integer or SIMD variants (in `jidctint.c`, `jidctfst.c`, etc.) may be compiled instead on platforms where float hardware is weak.

### 2. **Separable 2D Transform (Column-Row Decomposition)**
Lines 86–189 (Pass 1 columns) and 199–244 (Pass 2 rows) implement the same butterfly computations twice. Why not a direct 2D algorithm?
- **Efficiency:** Separable reduces multiply count: an 8-point 1D IDCT is ~5 multiplies; naive 2D is 64×. Two passes of 8 columns + 8 rows = 80 operations.
- **Architectural fit:** Modern CPUs cache-friendly; data flows linearly through memory.
- **Simplicity:** Single DCT code repeated, vs. complex 2D indexing.

### 3. **Sparsity Exploitation (Pass 1 Only)**
Lines 105–121 detect when all AC coefficients are zero in a column (common post-quantization). The output is then uniform (just the DC value scaled), skipping ~90% of the butterfly math. Pass 2 skips this check (lines 195–197 comment):
- **Why:** After Pass 1, most columns have introduced nonzero AC terms (typically only 5–10% of rows remain sparse).
- **Trade-off:** Float equality test is costlier than savings on sparse rows.

### 4. **Fixed-Size Block & Compile-Time Assertion**
Lines 53–56: deliberate syntax error if `DCTSIZE != 8`. This ensures:
- No silent miscompilation if JPEG block size changes.
- All loop unrolling and constants assume exactly 64 coefficients.
- Architectural necessity: JPEG standard mandates 8×8 MCUs.

### 5. **Intermediate Workspace Pattern**
`FAST_FLOAT workspace[DCTSIZE2]` (line 89) decouples passes:
- Pass 1 outputs floats; Pass 2 reads floats, then casts to `INT32` and descales.
- Avoids range-limiting in Pass 1, then discovering saturation in Pass 2.
- Stack-allocated (~256 bytes per invocation); no heap fragmentation.

## Data Flow Through This File

```
Input: JCOEFPTR coef_block (64 JCOEF values, quantized)
       FLOAT_MULT_TYPE *quantptr (64 dequant multipliers from compptr->dct_table)
       j_decompress_ptr cinfo (for sample_range_limit lookup table)

Pass 1 (lines 86–189):
  For each of 8 columns:
    • Check if AC coefficients (indices 8,16,...,56) are all zero (bitwise OR)
    • If yes: dcval = DEQUANTIZE(coef[0], quantptr[0]); fill column with dcval
    • If no:
      - Dequantize even indices (0,2,4,6) → tmp0,tmp1,tmp2,tmp3
      - Apply AA&N even-part butterfly → tmp0..tmp3, tmp10..tmp13
      - Dequantize odd indices (1,3,5,7) → tmp4..tmp7
      - Apply AA&N odd-part butterfly with magic constants (√2, 2cos(π/8), etc.) → z5,z10..z13
      - Merge even+odd → wsptr[0..7] in workspace

Pass 2 (lines 193–244):
  For each of 8 rows (reading from workspace):
    • Even-part butterfly (no dequantize; already done in Pass 1)
    • Odd-part butterfly
    • Merge to tmp0..tmp7
    • Final output stage: 
      - Compute tmp0+tmp7, tmp0-tmp7, etc. (8 intermediate values)
      - Cast to INT32
      - DESCALE by 3 (divide by 8, with rounding) to undo scaling from both passes
      - Index into range_limit lookup table
      - Write to outptr[0..7]

Output: JSAMPARRAY output_buf[output_col..output_col+7] (8 rows of 8 JSAMPLE values)
```

## Learning Notes

### Algorithmic Context
- **AA&N (Arai, Agui, Nakajima) Algorithm:** Canonical DCT optimization reducing multiplies from 11 to 5. The constants in lines 150, 162, 166–167 derive from their 1988 paper and appear in the JPEG standard reference implementations. Modern engines often use this or Loeffler's variant.
- **Why not FFT?** An 8-point FFT would use fewer operations if implemented efficiently, but DCT has simpler structure (real-only, symmetric); AA&N exploits this better.

### Era-Specific Patterns
1. **No SIMD:** This code predates widespread SIMD. Modern variants (libjpeg-turbo, libvpx) use SSE/AVX/NEON for 16+ parallel block processing.
2. **No GPU acceleration:** Textures are decompressed on CPU. Modern engines may use video APIs or compute shaders for JPEG decoding.
3. **Offline vs. Online:** This assumes decompression happens at load time, not streamed during gameplay. That's sensible for Q3A's asset model.

### Signal Processing Insights
- **Inverse transform:** DCT → pixel domain. The forward DCT (encoder) is mathematically the transpose; clever reuse of butterfly code.
- **Quantization → sparsity:** Quantization tables zero out high frequencies; columns with all AC zeros are thus common.
- **Range limiting:** Floating-point rounding can produce 256 or −1; the `range_limit` table clamps to 0–255 and prevents artifacts.

### Cross-Cutting Concerns Not Visible in First-Pass
- **Threading:** The function is **stateless**; multiple threads can decompress different textures simultaneously if `jddctmgr.c` supports it (likely not in Q3A, but the design allows it).
- **Texture streaming:** If the renderer supported progressive JPEG display, this function would be called repeatedly per MCU as data arrives.
- **JPEG profile diversity:** The file's presence alongside integer IDCT variants (`jidctint.c`) shows that the JPEG library supports multiple profiles. Q3A likely compiles both and lets `jddctmgr.c` choose at runtime based on `cinfo->do_fancy_upsampling` or platform capabilities.

## Potential Issues

1. **Hardcoded float constants** — The magic numbers (1.414213562 ≈ √2, 1.847759065 ≈ 2cos(π/8)) are float approximations. For JPEG (lossy), this is fine; for lossless or scientific use, higher precision would be needed.

2. **No input validation** — The function assumes `coef_block` and `quantptr` point to valid 64-element arrays. Invalid pointers or sizes will corrupt the workspace or output buffer. Caller bears responsibility.

3. **Undefined `SHIFT_TEMPS`** — Line 89 declares `SHIFT_TEMPS`, which must be defined in `jdct.h` (likely architecture-specific register variables for RISC platforms). If missing, compilation fails silently or produces inefficient code.

4. **Pass 2 sparsity decision** — The comment (lines 195–197) justifies skipping the zero-row check in Pass 2. This is a good heuristic, but on paths with many zero rows (e.g., highly quantized or synthetic images), it could waste CPU. A conditional check might be faster; this represents a micro-optimization trade-off.

5. **CENTERJSAMPLE offset** — The `range_limit` table has an implicit offset of `CENTERJSAMPLE` (likely 128). If `jdmaster.c` fails to populate it correctly, clamping will be wrong. Not strictly a bug in this file, but a subtle coupling.
