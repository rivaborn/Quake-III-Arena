# code/jpeg-6/jdct.h — Enhanced Analysis

## Architectural Role
This file is a **private algorithm dispatch interface** within the JPEG library's codec layer, consumed exclusively by the renderer's texture-loading subsystem (`code/renderer/tr_image.c` → JPEG loader → jcdctmgr.c/jddctmgr.c). It decouples algorithm selection from implementation, allowing the JPEG library to swap between slow-integer, fast-integer, and floating-point DCT/IDCT strategies *without recompiling algorithm managers* — a critical portability pattern in 1990s C libraries where architectural flexibility had to coexist with compiler limitations and hardware variance.

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c** (indirectly, via JPEG decoder): Calls JPEG library entry points, which eventually route through these DCT routines during decompression
- **code/jpeg-6/jcdctmgr.c**: DCT manager for compression; selects and caches forward DCT function pointers based on quality settings
- **code/jpeg-6/jddctmgr.c**: IDCT manager for decompression; selects inverse DCT routine and manages dequantization table formats
- **code/jpeg-6/jf*.c, ji*.c**: Individual algorithm implementations (islow, ifast, float variants); they consume the macro definitions and function pointer typedefs

### Outgoing (what this file depends on)
- **code/jpeg-6/jmorecfg.h** (configuration/platform macros): `BITS_IN_JSAMPLE`, `INT32`, `INT16`, `MULTIPLIER`, `FAST_FLOAT`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- **code/jpeg-6/jpegint.h** (internal JPEG API): `JMETHOD`, `JPP`, `EXTERN` macro definitions; declares `inverse_DCT_method_ptr` typedef
- Platform-specific **RIGHT_SHIFT** macro: Must be defined in config header; governs `DESCALE` signed-division rounding direction

## Design Patterns & Rationale

**Pattern: Function-pointer dispatch with algorithm-specific multiplier types**
- Why: The JPEG spec allows multiple trade-offs (accuracy vs. speed). Each algorithm implementation optimizes differently: `islow` uses `MULTIPLIER` (short), `ifast` uses 16-bit scaled factors on 8-bit samples, and `float` uses native floating-point. By parameterizing `IFAST_SCALE_BITS` and `*_MULT_TYPE`, the header lets each manager choose the right quantization table format *once during initialization* rather than branching per-block.

**Pattern: Compile-time linker portability hacks**
- The `NEED_SHORT_EXTERNAL_NAMES` block is a defensive **name-mangling layer** for platforms (e.g., old mainframes, embedded systems) with linkers that truncated identifiers or had limited symbol tables. This allows shipping the same .c files everywhere without preprocessor hell in the implementations.

**Pattern: Fixed-point arithmetic macros**
- `FIX()`, `DESCALE()`, `MULTIPLY16C16()` abstract **platform-specific optimization choices**. On a 1990s machine, a hand-crafted 16×16→32 multiply could be 10x faster than a full 32×32 multiply. Modern compilers optimize these away; back then, the casts and macro guards were *essential* performance tuning. The `SHORTxSHORT_32` ifdef branches reflect empirical per-platform profiling.

**Tradeoff: Scaled outputs over normalized ones**
- Forward DCT outputs are scaled up by 8×. This **defers division**, improving integer-arithmetic precision and letting quantization code do all scaling in one step. Modern floating-point engines skip this; fixed-point engines rely on it.

## Data Flow Through This File

1. **Compression path** (cgame writes pixel data):
   - `tr_image.c` loads JPEG via `ri.LoadJPEG()` 
   - JPEG decompressor initializes via `jddctmgr.c` (picks IDCT variant, allocates dequant tables with `IFAST_MULT_TYPE` or `FLOAT_MULT_TYPE`)
   - For each 8×8 MCU block: calls `jpeg_idct_ifast()` or `jpeg_idct_float()` with `compptr->dct_table` multiplier array
   - `jpeg_idct_ifast()` reads quantized coefficients, applies `dct_table[coeff] << IFAST_SCALE_BITS`, inverse-transforms, range-limits via `IDCT_range_limit()` table, writes output samples

2. **Decompression path** (server encodes map geometry):
   - Server-side game VM could theoretically use JPEG compression for streaming data (not typical in Quake III)
   - If used: `jcdctmgr.c` picks `jpeg_fdct_ifast()` or `jpeg_fdct_float()`, applies in-place on 64-element DCTELEM buffer, outputs scaled coefficients for quantization

## Learning Notes

**Idiomatic to JPEG era (mid-1990s), differs from modern engines:**
- No **SIMD intrinsics**: All math is scalar; relies on compiler autovectorization (didn't exist then) or hand-written assembly in separate `*.s` files
- No **memory layout optimization**: DCTELEM buffers are row-major, not tiled; cache-unfriendly by modern standards
- **Explicit platform configuration**: Every optimization hypothesis (`SHORTxSHORT_32`, `MULTIPLY16V16` fallback) is a compile-time knob, not a runtime adaptive choice
- **No error recovery**: Corrupt quantization tables or coefficient blocks will corrupt output silently; no bounds checks

**Connections to broader game engine concepts:**
- **Shader system analogy**: Like shader backends (ARB vs. GLSL), this file lets the JPEG library choose *algorithm variant* once and commit, rather than branching per-block
- **Resource streaming**: Texture decompression is on the critical path during level load; the speed difference between `ifast` and `islow` could be seconds
- **Quantization separation**: Mirrors modern GPU architecture — DCT is fixed, quantization is data-dependent and deferred. Back then, this was a memory-bandwidth optimization; today it's how GPUs partition compute

## Potential Issues

None clearly inferable from the file alone, but context-aware notes:

- **Endianness assumptions**: The `AAS_SwapAASData` cross-reference in the context excerpt suggests byte-swapping routines exist elsewhere. JPEG DCT/IDCT on big-endian hardware (PPC, MIPS) could silently produce wrong results if dequantization table endianness is mishandled. The header doesn't document this.
- **BITS_IN_JSAMPLE == 12 path**: Support for 12-bit JPEG is declared but rarely tested in game engines. A rare platform-specific compilation could leave latent bugs.
