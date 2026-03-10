# libs/jpeg6/jfdctflt.cpp — Enhanced Analysis

## Architectural Role

This file implements the floating-point forward DCT for the JPEG decompression pipeline integrated into Quake III's renderer. The renderer's texture-loading subsystem (`tr_image.c`) depends on libjpeg-6's complete decompression stack, of which this DCT is the core inverse-domain frequency transformation. The choice of floating-point over fixed-point arithmetic here represents a deliberate architectural tradeoff: trading throughput for numerical accuracy and simplified cross-platform behavior, since the DCT output feeds downstream quantization and color-space conversion in the JPEG decoder.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture loading** (`code/renderer/tr_image.c`): Calls into `jload.c` (libjpeg JPEG decode entry point), which transitively invokes `jpeg_fdct_float()` during baseline DCT coefficient recovery
- **Offline tools** (bspc, q3map, q3radiant): May also link libjpeg for texture preview or asset validation, though primary runtime path is renderer-only

### Outgoing (what this file depends on)
- **libjpeg infrastructure** (`jdct.h`, `jpeglib.h`, `jinclude.h`): Macro definitions (`DCTSIZE`, `FAST_FLOAT`), quantization table references, and the DCT function-pointer dispatch table
- **Compile-time DCT selection** (`#ifdef DCT_FLOAT_SUPPORTED`): The build system chooses this implementation over integer or multiplierless variants

## Design Patterns & Rationale

**Separable 2D Transform:**  
The code implements the standard factorization of 8×8 DCT into two sequential 1D DCT passes (rows then columns), exploiting the separability property. This reduces complexity from O(n⁴) to O(2n²) and allows precise row/column indexing without a transposition buffer—critical for embedded systems.

**AA&N (Arai–Agui–Nakajima) Algorithm:**  
The implementation reduces the DCT from 11 multiplies (naive) to 5 multiplies + 29 additions by carefully scheduling operations so scaling multiplies fold into downstream quantization. Intermediate variables (`z1`–`z5`, `z11`–`z13`) cache sub-expressions across the even/odd butterfly structure (phases 2–6).

**Floating-Point Precision vs. Portability:**  
By using `FAST_FLOAT` (typically `float`) instead of fixed-point integer math, the code accepts:
- Platform-dependent roundoff behavior (mentioned in header)
- Potential accuracy drift across CPU architectures (x86 80-bit vs ARMv7 32-bit paths)
- Better numerical stability for accumulating weighted sums in the odd-part rotator

This is acceptable because texture lossy compression (JPEG itself) already introduces quantization error far exceeding floating-point precision limits.

## Data Flow Through This File

**Input:** 64 floats in row-major order (8×8 DCT block), representing raw pixel intensity or chroma after color-space decomposition  
**Process:**
1. **Row pass** (lines 72–117): For each of 8 rows, apply 1D DCT using AA&N factorization; output DC (coeff [0]) and AC (coeffs [1–7]) in place
2. **Column pass** (lines 119–168): Treat the row-pass results as 8 column vectors; apply identical 1D DCT; write final frequency coefficients in-place

**Output:** 64 floats in frequency domain, ready for JPEG quantization table division and entropy coding  
**State mutation:** The `FAST_FLOAT * data` pointer is modified in-place; no intermediate buffering.

## Learning Notes

**Modern engine pattern divergence:**  
Early-2000s engines often embed legacy third-party libraries (here, libjpeg-6a ca. 1998) with minimal wrapping. The monolithic `code/jpeg-6/` tree (not decomposed into a shared system library) reflects an era before dynamic system libs were reliably available. Modern engines typically dlopen() system JPEG or AVIF decoders at runtime.

**DCT in the broader pipeline:**  
Developers unfamiliar with JPEG encoding should understand the data flow: **Raw JPEG bitstream** → `jdatasrc` (**bitstream buffer**) → `jdhuff` (**Huffman decode**) → **Quantized coefficients** → **This DCT (fdct or idct inverse)** → `jdcolor` (**color-space conversion**) → **Raw RGB pixels** → `tr_image` (**GPU upload**). The fdct here is the *inverse* DCT (IDCT), despite the misleading name; it reconstructs spatial pixels from frequency coefficients.

**Idiomatic to this era:**  
- Macro-heavy configuration (`DCTSIZE != 8` compile-time check is intentional; Q3 forbids dynamic block sizes)
- In-place algorithms to minimize hunk allocations (critical on PS2, Dreamcast memory constraints)
- No SIMD: the code runs scalar FPU operations; MMX/SSE variants exist elsewhere in the codebase but not here

## Potential Issues

- **Floating-point consistency:** If the same JPEG is decoded on different architectures (e.g., during demo playback across PC/console), pixel values may diverge slightly due to accumulated rounding. Modern engines use fixed-point DCT or strict bit-level reproducibility guarantees for multiplayer demos.
- **No `#pragma pack` or alignment hints:** The rowwise loop advances `dataptr += DCTSIZE` assuming 8-element strides; if `FAST_FLOAT` ever changes from 4 bytes (float) to 8 bytes (double), stride calculations silently break.
