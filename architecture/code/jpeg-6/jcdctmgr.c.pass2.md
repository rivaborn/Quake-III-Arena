# code/jpeg-6/jcdctmgr.c — Enhanced Analysis

## Architectural Role

This file is the **strategy dispatcher and precomputation coordinator** for the vendored IJG libjpeg-6 JPEG compression library, used exclusively by the renderer's texture-loading pipeline (`code/renderer/tr_image.c` → `jload.c`). It sits at the boundary between the engine and a third-party library: while called transitively during texture asset preparation, it remains entirely **decoupled from core engine subsystems** (qcommon, server, game VM). Its role is to absorb DCT algorithm selection and divisor-table precomputation, isolating callers from the complexity of multiple fixed-point and floating-point implementations.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** → calls `LoadJPG` in `jload.c` → invokes `start_pass_fdctmgr` during compression pass initialization
- Other libjpeg-6 modules (entropy encoding layer) consume the quantized DCT coefficients produced by `forward_DCT_float`
- The JPEG library lifecycle: `jinit_forward_dct` called once at compressor initialization; `start_pass_fdctmgr` called per-pass; `forward_DCT_float` called per-block in the innermost encode loop

### Outgoing (what this file depends on)
- **Within libjpeg-6**: calls delegated DCT routines (`jpeg_fdct_islow`, `jpeg_fdct_ifast`, `jpeg_fdct_float`) defined in sibling `.c` files
- **Global JPEG state**: `jpeg_zigzag_order` table from IJG library initialization
- **Memory allocator**: `cinfo->mem->alloc_small` for persistent divisor table allocation
- **Error handling**: `ERREXIT` / `ERREXIT1` macros from `jerror.h` (IJG error context)
- No dependencies on any Q3A engine subsystems; completely isolated

## Design Patterns & Rationale

**Strategy Pattern**: DCT algorithm selection is deferred to runtime (`jinit_forward_dct`) based on `cinfo->dct_method`, allowing the same encoder interface to dispatch to islow/ifast/float implementations without conditional branches in the hot path.

**Precomputation for Hot-Path Optimization**: Divisor tables are built once per quantization table per pass (`start_pass_fdctmgr`), then cached for repeated use in `forward_DCT_float`. This trades **upfront allocation overhead** for **elimination of division in the per-coefficient loop** — a classic micro-optimization for 1990s hardware where division was expensive.

**Reciprocal Trick (Float Path)**: Rather than storing divisor directly, the float implementation stores `1/divisor`. This allows the innermost loop to use multiplication (`temp * divisors[i]`) instead of division, which is both faster and sidesteps portability issues with C's unspecified rounding direction for negative quotients. The integer paths use explicit rounding logic (`qval >> 1` and conditional branching).

**Architectural Layering**: This file is the **management layer** that shields callers from algorithm-selection complexity. The actual DCT compute (`jpeg_fdct_*`) is separated into dedicated modules, following IJG's modular architecture.

**Conditional Compilation**: The integer `forward_DCT` function is **compiled out** (`#if 0`), suggesting this build uses only the float path. This is unusual for a game engine and may indicate the codebase was ported from a platform with strong FP support or the integer path was disabled after profiling.

## Data Flow Through This File

```
Texture asset (pixel data)
    ↓ [tr_image.c → jload.c → Compress JPEG]
    ↓ [cinfo->fdct = jinit_forward_dct() allocates my_fdct_controller]
    ↓ [start_pass_fdctmgr() validates quantization tables, precomputes divisor tables]
    ↓ [for each 8×8 block]
       ├─ Load samples + unsigned→signed conversion (CENTERJSAMPLE bias removal)
       ├─ Call fdct->do_float_dct(workspace) → forward DCT in-place
       ├─ Quantize/descale: output[i] = round(dct_coeff[i] * reciprocal_divisor[i])
       └─ Write JCOEF to coef_blocks[i]
    ↓
Quantized DCT coefficients → entropy encoder → JPEG bitstream
```

**State transitions**: 
- `NULL` divisor pointers (init) → allocated per quantization table per pass → reused across blocks
- Floating-point workspace is stack-allocated per block (ephemeral; 64 × FAST_FLOAT)

## Learning Notes

**Idiomatic to this era**: 
- The aggressive micro-optimization (reciprocal trick, loop unrolling, fixed-point precomputation) reflects 1990s priorities: FPUs were still luxury items, division was slow, cache-line thrashing was common. Modern engines would rely on SIMD intrinsics and specialized hardware codecs.
- **Hardcoded DCTSIZE == 8**: The code assumes 8×8 DCT blocks (JPEG standard). Modern image codecs support variable block sizes.
- **Three independent implementations** (islow, ifast, float) for what is fundamentally the same algorithm reflects the era's hardware diversity (fixed-point-only CPUs, slow FPU, no SIMD). A modern version would likely use a single SIMD path or offload to GPU.

**Not an ECS or scene-graph pattern**: This is pure procedural utility code — no entities, no hierarchies, no late binding. It directly manipulates arrays and function pointers.

**Texture pipeline position**: This is part of **offline asset preparation**, not real-time rendering. The fact that Q3A can dynamically load JPEG textures (rather than requiring pre-processed BC/DXT) is architecturally interesting but unusual for a 1999 engine; most competitors shipped with offline toolchains that baked textures into proprietary formats.

## Potential Issues

1. **Floating-point precision**: The rounding trick `(int)(temp + 16384.5) - 16384` assumes that `FAST_FLOAT` can safely represent ±16K without losing precision. On platforms where `FAST_FLOAT` is single-precision float, coefficients close to ±2^15 could round incorrectly, but this is likely acceptable for lossy compression.

2. **Divisor table reallocation**: `start_pass_fdctmgr` checks `if (fdct->divisors[qtblno] == NULL)` before allocating, meaning tables persist across passes. If a quantization table is modified mid-session (unlikely), old divisor values would be stale.

3. **No bounds checking on `compptr->quant_tbl_no`**: The code validates the index with `ERREXIT1`, which is correct, but earlier code (`start_pass_fdctmgr`) trusts the component info is well-formed. A corrupted BSP or malformed entity data could theoretically cause out-of-bounds access before the check fires.

4. **Compiled-out integer path**: The `#if 0` on `forward_DCT` means the integer quantization logic (rounding, fast-divide optimization) is dead code. If the float path is disabled at compile time, the encoder would fail. This suggests **fragile conditional compilation** — a single-platform binary.
