# libs/jpeg6/jdct.h — Enhanced Analysis

## Architectural Role

This file defines the **private algorithmic interface** between DCT (Discrete Cosine Transform) strategy implementations and their manager/dispatcher functions. It sits at the algorithmic abstraction layer within the vendored libjpeg-6 library, isolated from the public API. The Renderer subsystem (`code/renderer/tr_image.c`) calls into the JPEG library for texture decompression, but does not directly reference this file—instead, it uses the public `jload.c` entry point, which internally selects and dispatches to these forward/inverse DCT routines based on the image's bit-depth and compilation flags.

## Key Cross-References

### Incoming (who depends on this file)
- **jcdctmgr.c** (IJG forward DCT manager) — selects and calls `jpeg_fdct_islow`, `jpeg_fdct_ifast`, or `jpeg_fdct_float` based on quantization speed hints
- **jddctmgr.c** (IJG inverse DCT manager) — calls `jpeg_idct_islow`, `jpeg_idct_ifast`, `jpeg_idct_float`, and size-specific variants (`_4x4`, `_2x2`, `_1x1`) for partial block reconstruction
- **Individual DCT implementation files** (jfdctint.c, jfdctfst.c, jfdctflt.c, jidctint.c, jidctfst.c, jidctflt.c, etc.) — provide extern definitions for their respective functions

### Outgoing (what this file depends on)
- **jpegint.h** (IJG internal type definitions) — defines `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`
- **jmorecfg.h** — defines `INT32`, `MULTIPLIER`, `FAST_FLOAT` conditioned on compilation flags
- **jconfig.h** — defines `BITS_IN_JSAMPLE` (8 or 12, set at configure time)
- No external dependencies beyond libjpeg's own internal ecosystem

## Design Patterns & Rationale

**Strategy Pattern** — The six DCT algorithm variants (islow, ifast, float, plus IDCT size variants) are pluggable strategies selected at runtime. The manager code (`jcdctmgr.c`/`jddctmgr.c`) never hard-codes which implementation to use; instead, it receives a function pointer and invokes it generically. This allows the same manager code to dispatch across performance tiers.

**Abstraction Layer** — Isolates quantization/dequantization concerns (which are manager responsibilities) from DCT computation. The interface contract specifies:
- **Input/output semantics**: what scale factors are applied, whether data is signed/unsigned, and precision guarantees
- **Fixed-point conventions**: all DCT outputs are scaled by 8× for 8-bit data; IDCT inputs are dequantized coefficients that the algorithm must range-limit to `0..MAXJSAMPLE`

**Macro-Based Fixed-Point Arithmetic** — The `FIX()`, `DESCALE()`, `MULTIPLY16C16()` macros were critical optimizations on 1990s CPUs where integer multiply-accumulate was faster than floating-point. The macros allow compile-time constant folding for scale factors while deferring shifts to runtime for variable data.

**Conditional Compilation for Bit-Depth** — `DCTELEM` is `int` for 8-bit JPEG (faster) but `INT32` for 12-bit (to prevent overflow in intermediate math). Similarly, `IFAST_SCALE_BITS` is platform-dependent (2 for 8-bit, 13 for 12-bit), optimizing multiplier table size without altering the algorithm.

## Data Flow Through This File

1. **Forward DCT pipeline:**
   - Input: source samples (signed, centered around zero) + work buffer
   - Manager (`jcdctmgr.c`) → selects forward DCT variant → calls `jpeg_fdct_*`
   - Algorithm in-place transforms work buffer, scales output by 8×
   - Output: DCT coefficients (±8K for 8-bit, ±128K for 12-bit)
   - Quantization manager then dequantizes coefficients

2. **Inverse DCT pipeline:**
   - Input: quantized DCT coefficients + dequantization multiplier table
   - Manager (`jddctmgr.c`) → selects inverse IDCT variant (size-dependent) → calls `jpeg_idct_*`
   - Algorithm dequantizes, computes IDCT, range-limits to `0..MAXJSAMPLE`
   - Output: reconstructed samples placed at specified row/column in output array
   - Renderer/texture system then uses reconstructed samples for GPU upload

3. **Scale factor invariants:**
   - Forward: input ±CENTERJSAMPLE → output ±8K requires 3 bits headroom; preserves precision for quantization
   - Inverse: dequantized coefficients → output `0..255` requires range-limit lookup table (`IDCT_range_limit` macro) to bulletproof against corrupt input

## Learning Notes

- **Late-90s portable C optimization**: The fixed-point arithmetic macros and conditional platform-specific multiplier types reflect a design era when every CPU variant (386, PPC, SPARC) had different performance tradeoffs between integer and floating-point paths. Modern compilers/CPUs have made this less critical, but the approach is historically instructive.

- **Linker compatibility workarounds** (`NEED_SHORT_EXTERNAL_NAMES`): On some ancient Unix variants and embedded systems, linkers truncated or folded symbol names. The short names (`jFDislow` instead of `jpeg_fdct_islow`) were a pragmatic solution; modern linkers don't need this.

- **Idiomatic to this era**: The extern declarations directly in the algorithm interface (rather than a separate dispatcher typedef) assume static linking or weak symbols. Modern engines use explicit vtable/callback function pointers to decouple algorithm selection from compilation.

- **Connection to texture pipeline**: This DCT header is invisible to the Renderer; `tr_image.c` only knows about `jload.c` (the public JPEG loader). The DCT strategy selection happens entirely within libjpeg's internal architecture, making it a good example of **internal modularity within a vendored library** that the host engine doesn't need to reason about.

## Potential Issues

- **Fixed-point precision edge cases**: The `FIX(x)` macro uses naive rounding (`+ 0.5`). For some irrational constants (e.g., cos coefficients), this could accumulate error across multiple DCT stages, especially in the `islow` (integer-only) implementation.

- **Range-limit table assumption**: The `IDCT_range_limit` macro assumes `cinfo->sample_range_limit` has been pre-allocated and initialized by `prepare_range_limit_table()`. If that initialization is skipped or fails, IDCT will dereference garbage; no bounds checking is present.

- **No validation of bit-depth consistency**: If `BITS_IN_JSAMPLE` is mismatched between compile-time configuration and runtime image format, `DCTELEM` width may overflow without warning. This is historically a source of silent corruption on heterogeneous build environments.
