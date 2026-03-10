# libs/jpeg6/jidctflt.cpp — Enhanced Analysis

## Architectural Role

This file implements the floating-point inverse DCT (Discrete Cosine Transform) component of the vendored IJG libjpeg-6 library, which sits on the texture loading path of the renderer. When the renderer loads a JPEG texture file (`code/renderer/tr_image.c`), the full JPEG decompression pipeline executes: entropy decoding → dequantization → **IDCT (this file)** → color upsampling → RGB conversion → OpenGL upload. The IDCT is the computational heart of JPEG decompression, transforming quantized 8×8 frequency-domain blocks back to spatial-domain pixels.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** calls IJG libjpeg decompression functions during `R_LoadJPG`, which chains to this IDCT for each 8×8 block in the encoded image
- **Renderer subsystem** (via `LoadImage` / `R_CreateImage`) depends on texture loading; JPEG is one of the supported formats
- No explicit function pointers; IDCT is invoked indirectly through the standard libjpeg decompression API

### Outgoing (what this file depends on)
- **IJG libjpeg headers only**: `jinclude.h`, `jpeglib.h`, `jdct.h` for type definitions and macros
- **Zero runtime engine dependencies**: This is pure mathematical computation with no calls to qcommon, renderer, or engine code
- **Macro-driven platform abstraction**: `FAST_FLOAT`, `DEQUANTIZE`, `DESCALE`, `SHIFT_TEMPS` allow compile-time configuration; actual types/implementations come from `jdct.h`

## Design Patterns & Rationale

**Two-Pass 1D Decomposition**: The 2D 8×8 IDCT is decomposed into two sequential 1D passes (columns, then rows). This reduces complexity from O(n⁴) to O(n³) and improves cache locality compared to direct 2D algorithms.

**Sparse Data Fast-Path** (lines 102–115): JPEG quantization typically zeros out high-frequency AC coefficients. The code detects all-zero AC terms (`inptr[DCTSIZE*1] | ... | inptr[DCTSIZE*7]`) and replaces the full IDCT computation with a simple broadcast of the DC value. This optimization applies to ~50% of blocks in typical images and was essential for acceptable software JPEG decompression performance in the 1990s–2000s era.

**AA&N Algorithm**: Uses the Arai, Agui, Nakajima scaled DCT method (described in Pennebaker & Mitchell JPEG textbook, figure 4-8). This specific decomposition minimizes the number of multiplies (only 5 per column/row pass vs. 11+ for naive DCT) by folding dequantization multipliers into the fixed cosine scaling factors. The magic numbers (e.g., `1.414213562` = √2) arise from this mathematical decomposition.

**Range-Limiting Lookup Table** (lines 209–220): Final output is clamped via `range_limit[index & RANGE_MASK]`, a precomputed lookup table. This avoids conditional branches and padding logic; the table is populated by `IDCT_range_limit()` at initialization.

## Data Flow Through This File

**Input**: 
- `coef_block`: 64 quantized DCT coefficients (array of `JCOEF`)
- `compptr->dct_table`: Per-component dequantization multipliers
- `output_buf[output_col]`: Destination for 8×8 output block

**Processing**:
1. **Pass 1 (lines 99–188)**: For each of 8 columns:
   - Dequantize all 8 coefficients via `DEQUANTIZE` macro
   - Check sparse-data optimization (all AC terms zero?)
   - If not sparse: compute even/odd parts of 1D IDCT using AA&N factorization
   - Store 8 floating-point samples into `workspace[DCTSIZE*0..7]` (row-major per-column layout)

2. **Pass 2 (lines 191–220)**: For each of 8 rows in workspace:
   - Load 8 workspace samples (already dequantized and column-IDCTed)
   - Compute even/odd parts (identical math to Pass 1, but no dequantization)
   - **Descale by 8** (divide-by-2³) and cast to integer to counter the accumulated scaling from two passes
   - Range-limit to [0, 255] and store into output buffer

**Output**: 64 unsigned 8-bit samples, one 8×8 block of reconstructed image pixels.

## Learning Notes

**What This File Teaches**:
- The IJG libjpeg design pattern: self-contained, compile-time configurable IDCT implementations (integer, fixed-point, floating-point) selected at build time
- The JPEG decompression pipeline: entropy decode → dequantize → IDCT → upsample → color-transform
- Sparse-data optimization techniques: how to exploit the structure of quantized DCT blocks
- Mathematical signal processing: the AA&N algorithm is a textbook example of optimizing linear transforms by minimizing arithmetic operations

**Era-Specific Context**:
- This code (IJG libjpeg-6) dates from ~1998–2002, when software JPEG decompression was the standard path on desktop CPUs
- Floating-point IDCT was attractive post-Pentium II (hardware FP pipelines became efficient)
- By Q3A's 2005 release, this was already legacy; modern engines were moving toward GPU texture decompression
- The sparse-data optimization was *critical* for real-time software decompression in the late 1990s

**Modern Engine Contrast**: Contemporary engines (2010+) typically:
- Load JPEGs via GPUs using hardware decoders or pre-converted uncompressed/BC-compressed textures
- Avoid software decompression in the hot path
- Use more aggressive compression (JPEG 2000, ASTC, WebP) with different IDCT variants

## Potential Issues

1. **Non-Deterministic Output**: Per the header comment (line 18), floating-point roundoff behavior varies by CPU architecture. This means JPEG decompression may produce slightly different pixel values on x86 vs. PowerPC, which violates byte-exact reproducibility. For texture assets this is cosmetic, but it breaks strict bit-exact demo recording if enabled.

2. **No SIMD Acceleration**: The code is pure scalar floating-point. No SSE/AVX variants exist in this library distribution. The `code/unix/matha.s` and similar assembly files suggest the engine uses SIMD elsewhere, but this IDCT is scalar-only.

3. **Deprecated Algorithm**: The AA&N algorithm (1989) has been superseded by more efficient integer-only fast DCTs (e.g., LLM algorithm used in libjpeg-turbo, which Q3A does not use). Re-encoding all textures with a modern compressor would reduce file size and load time.

4. **Workspace Stack Allocation** (line 84): `FAST_FLOAT workspace[DCTSIZE2]` (64 floats = 256 bytes) is allocated on the stack per-block. On systems with small stack, many concurrent texture loads could cause stack overflow. Use is safe in practice because JPEG loading is single-threaded, but note the assumption.
