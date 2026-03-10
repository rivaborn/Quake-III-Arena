# code/jpeg-6/jccolor.c — Enhanced Analysis

## Architectural Role
This file implements the input colorspace conversion pipeline for the IJG JPEG **compressor**. However, the Quake III engine uses vendored libjpeg-6 exclusively for **decompression** (loading textures via `renderer/tr_image.c` → `jload.c`), not compression. Consequently, this module has **no active runtime role** in the engine—it's included for build completeness but unused. The corresponding decompression-side counterpart (`jdcolor.c`) is the relevant module for texture asset loading.

## Key Cross-References

### Incoming (who depends on this file)
- **None in the runtime engine.** This compression module is not called by any subsystem (renderer, qcommon, client, or server).
- Included in `code/jpeg-6/` as part of the full IJG libjpeg library port, but the engine builds only the decompression path.

### Outgoing (what this file depends on)
- **`jpeglib.h` / `jinclude.h` / `jmorecfg.h`** — IJG public/internal API; defines all `J_COLOR_SPACE` enum values, `jpeg_color_converter` vtable, `MAXJSAMPLE`, `GETJSAMPLE`, `CENTERJSAMPLE`, `RGB_*` constants.
- **`cinfo->mem->alloc_small`** — Memory manager callback (internal to libjpeg); allocates fixed-size lookup tables from `JPOOL_IMAGE` pool.
- **No engine subsystem dependencies.** This is self-contained IJG code.

## Design Patterns & Rationale

### Fixed-Point Lookup Table Optimization
The core insight: Avoid per-pixel floating-point arithmetic by precomputing all coefficient×sample products into 256-entry (per channel) lookup tables. Each RGB sample (0–255) has a precomputed Y, Cb, Cr contribution stored in scaled integer form. The inner loop then reduces to:
- 3 table lookups + 2 adds + 1 right-shift per component
- No floating-point, no general division, no range-clipping

This was critical in the 1990s (when this code was written) for real-time compression on slower CPUs. The `SCALEBITS=16` choice gives ~4 decimal digits of precision while fitting comfortably in 32-bit integer math.

### Function-Pointer Dispatch Pattern
`jinit_color_converter` implements a classic **module initialization** pattern: validate inputs, then wire up the appropriate `(start_pass, color_convert)` function-pointer pair. This decouples the init decision tree from per-row execution, allowing tight inner loops without repeated conditionals.

### Tradeoff: Lookup Table Memory vs. Arithmetic
For 8-bit samples, the 8×256 table (8 KB) is trivial. For 12-bit samples, it scales to ~32 KB (acceptable per the comment). For 16-bit samples, a 8×65536 table would be ~2 MB—impractical for lossless storage, hence the code explicitly discourages 16-bit compression.

## Data Flow Through This File

**Initialization phase** (`jinit_color_converter`):
- Engine-side configuration is passed via `cinfo` (input/output colorspace, component counts)
- Allocate `my_color_converter` struct; store in `cinfo->cconvert`
- Validate compatibility matrix (e.g., JCS_RGB input → JCS_YCbCr output is legal)
- Assign function pointers; set `start_pass` to `rgb_ycc_start` if table initialization is needed

**First pass** (if `start_pass` is called):
- Populate the 8-section lookup table with precomputed coefficients and offsets
- Section boundaries defined by `R_Y_OFF`, `G_Y_OFF`, etc. macros—all fold into a single contiguous array

**Per-row conversion** (during compression):
- `color_convert` is called once per batch of input rows
- Input rows are interleaved (RGB, CMYK, or YCbCr per-pixel)
- Output is planar (separate buffers for Y, Cb, Cr or equivalent)
- For RGB→YCbCr: de-interleave and convert in a single pass

## Learning Notes

### Idiomatic to This Era
1. **Fixed-point integer math dominates.** No SIMD, no GPU compute; the `SCALEBITS` pattern is from before MMX.
2. **Monolithic table precomputation.** Modern code might generate tables dynamically or use bit-shifting tricks; here they're baked in at init.
3. **Planar vs. interleaved format mismatch.** Applications supply interleaved RGB; JPEG internally uses planar YCbCr. This file bridges that gap—a task that modern GPU codecs avoid by operating natively on one format.
4. **Aliasing optimization** (`R_CR_OFF == B_CB_OFF`): the FIX coefficients for R→Cr and B→Cb happen to be numerically identical (0.5), so the code reuses a single table section—a micro-optimization that surprises modern readers.

### Connection to Engine Concepts
- **Vendor library integration:** The engine treats libjpeg as a black-box library with minimal customization. Only decompression (`jdcolor.c` + `jdctmgr.c` + `jdcoefct.c` + etc.) is exercised; this compression module is dead code.
- **BSP color encoding:** Quake III's map/model textures are stored as JPEG-compressed assets in `.pk3` archives. The renderer loads them via `jload.c` (decompression), never writes them.
- **Format stability:** Choosing planar YCbCr internally (CCIR 601-1 coefficients) is a stable interchange format. All subsystems agree on `[0, MAXJSAMPLE]` range and `CENTERJSAMPLE` center offset.

## Potential Issues

### Dead Code (Non-Critical)
This entire file is unused in the runtime engine. Removing it would shrink the binary by ~15 KB and simplify builds, but it doesn't cause any malfunction because the compression entry points are never called. The presence of unused compression code is unusual for a **game engine** but consistent with including the full IJG library verbatim.

### Correctness: CENTERJSAMPLE Convention (Already Handled)
The comment at line ~28 explains the historical choice to use `CENTERJSAMPLE` rather than `MAXJSAMPLE/2` for Cb/Cr offsets. This ensures grayscale values map to exactly (Cb=0, Cr=0) rather than being off-by-0.5. The code correctly bakes this into the lookup tables, so no runtime error is possible.
