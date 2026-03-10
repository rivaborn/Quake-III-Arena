# code/jpeg-6/jdinput.c — Enhanced Analysis

## Architectural Role

This file implements the **state machine coordinator** for the JPEG decompression pipeline, managing the critical bifurcation between two phases: marker reading (structural metadata) and coefficient consumption (compressed image data). It sits at the intersection of the marker reader (`jdmarker.c`), entropy decoders (`jdhuff.c`/`jdphuff.c`), and coefficient controller (`jdcoefct.c`). In the Quake III renderer, this is invoked exclusively from the texture-loading path (`tr_image.c` → `jload.c` → libjpeg public API) to decompress JPEG images into raw pixel data for GPU upload; it has no interaction with game logic, networking, or bot subsystems.

## Key Cross-References

### Incoming (who depends on this file)
- **`jdapi.c`** (JPEG public C API): calls `jpeg_start_decompress`, which triggers the first `consume_input` dispatch into this module
- **`jdmaster.c`** (decompressor pipeline sequencer): calls `reset_input_controller`, `start_input_pass`, and indirectly via virtual function pointers
- **`jdcoefct.c`** (coefficient controller): provides the `consume_data` routine that alternates with `consume_markers` in the main loop
- **Renderer** (`code/renderer/tr_image.c` via `jload.c`): drives JPEG decompression as part of texture asset loading during map load and dynamic texture streaming

### Outgoing (what this file depends on)
- **`jdmarker.c`** (marker reader): called via `cinfo->marker->read_markers()` to parse JPEG markers (SOS, EOI, DQT, etc.)
- **`jdhuff.c` / `jdphuff.c`** (entropy decoders): initialized via `cinfo->entropy->start_pass()` for each scan
- **`jdcoefct.c`** (coefficient controller): initialized via `cinfo->coef->start_input_pass()`, provides data consumer callback
- **`jmemmgr.c`** (memory manager): allocates permanent and image-lifetime objects via `cinfo->mem->alloc_small()`
- **`jutils.c`** (utilities): `jdiv_round_up()` for ceiling division in MCU/component geometry calculations

## Design Patterns & Rationale

**Method Pointer / Virtual Function Pattern**: All inter-module communication uses function pointers (`cinfo->entropy->start_pass`, `cinfo->marker->read_markers`, etc.) rather than direct calls. This is classic pre-C++ OOP architecture (libjpeg predates C++), enabling swappable implementations (e.g., Huffman vs arithmetic entropy coding) without recompilation. The pattern is consistent across the entire libjpeg codebase.

**State Machine via Function Pointer Reassignment**: The core loop (`consume_input`) is a function pointer that switches between `consume_markers()` (inter-scan state) and `cinfo->coef->consume_data()` (intra-scan state). This avoids explicit state enums and allows the coefficient controller to fully encapsulate data-reading logic. Once a scan completes, `finish_input_pass()` reassigns `consume_input` back to `consume_markers()`.

**Lazy Initialization on First SOS**: The `initial_setup()` function defers all image geometry computation until the first SOS marker is encountered. This is economical because JPEG header parsing (dimensions, component count) is done separately by jdmarker, and expensive calculations (MCU counts, block dimensions) only happen once per decompression session.

**Multi-Scan Coordination via Quantization Table Latching**: The `latch_quant_tables()` function solves an important problem: in multi-scan JPEG (progressive or compound), the JPEG spec allows quantization table slots to be redefined between scans. To ensure correct dequantization of all components, this function **copies** each component's quantization table at the start of its first scan, preserving the exact table values that were current during that scan. Subsequent redefinitions don't affect already-latched tables. This is defensive design against pathological but technically valid JPEG streams.

**Transcoder vs Full-Decompressor Duality**: The code initializes `DCT_scaled_size = DCTSIZE` here as a fallback, but comments acknowledge that `jdmaster.c` will override it in normal decompression. This dual-path design supports JPEG transcoders (decode markers + re-encode without full inverse DCT), a space-saving optimization for tools like `cjpeg`.

## Data Flow Through This File

**Initialization Phase** (once at decompressor creation):
```
jinit_input_controller()
  ↳ allocate my_input_controller from JPOOL_PERMANENT
  ↳ wire method pointers: consume_input = consume_markers, 
                           reset_input_controller, start_input_pass, finish_input_pass
  ↳ set inheaders = TRUE
```

**Main Decompression Loop** (repeated calls from application):
```
consume_input() [currently consume_markers]
  ↳ read_markers() → parse SOI, DQT, SOF, DHT, etc. markers
  
  On JPEG_REACHED_SOS (Start of Scan):
    if first SOS:
      initial_setup()
        ↳ validate image dimensions, precision, component count
        ↳ compute max_h/v_samp_factor
        ↳ for each component: compute width_in_blocks, height_in_blocks, 
                              downsampled_width/height (sampling-aware geometry)
        ↳ compute total_iMCU_rows
        ↳ set has_multiple_scans flag
    else:
      start_input_pass() [called by jdmaster/application, not here]
  
  On JPEG_REACHED_SOS (not first):
    start_input_pass()
      ↳ per_scan_setup()
           → if noninterleaved: MCU_width=1, MCU_height=1 (simple case)
           → if interleaved: for each component, MCU_width = h_samp_factor,
                                                  MCU_height = v_samp_factor
                            → build MCU_membership[] array: maps each block 
                                position in an MCU to component index
      ↳ latch_quant_tables()
           → for each component in scan: copy quantization table to component-local copy
      ↳ entropy->start_pass()
      ↳ coef->start_input_pass()
      ↳ consume_input = coef->consume_data [switch to data phase]
  
  On JPEG_REACHED_EOI:
    set eoi_reached = TRUE
    if not in headers: clamp output_scan_number to input_scan_number

During Data Phase:
  consume_input() [currently coef->consume_data, provided by jdcoefct.c]
    ↳ reads huffman-coded MCU blocks
    ↳ when scan complete: coef calls finish_input_pass()
         → consume_input = consume_markers [switch back to marker phase]

On Reset (for reuse):
  reset_input_controller()
    ↳ reset consume_input to consume_markers
    ↳ err->reset_error_mgr() [clears accumulated error state]
    ↳ marker->reset_marker_reader()
    ↳ set coef_bits = NULL
```

**Key State Transitions**:
- `inheaders=TRUE` (initial) → `inheaders=FALSE` (after first SOS)
- `consume_input=consume_markers` (inter-scan) ↔ `consume_input=coef->consume_data` (intra-scan)
- `eoi_reached=FALSE` (initial) → `eoi_reached=TRUE` (on EOI marker) → read loop terminates

## Learning Notes

**Multi-Component Sampling in JPEG**: JPEG allows Y, Cb, Cr components to be subsampled at different rates (e.g., 4:2:0 means Cb/Cr at half resolution). The `per_scan_setup()` function computes the MCU ("Minimum Coding Unit") layout: an MCU is a composite tile containing all blocks for all components at a given sampling rate. For 4:2:0, a typical MCU is 16×16 pixels: four 8×8 Y blocks, one 8×8 Cb block, one 8×8 Cr block. The `MCU_membership[]` array tells the entropy decoder which component each block belongs to.

**Interleaved vs Non-Interleaved Scans**: 
- **Interleaved**: Multiple components encoded in the same scan, blocks interleaved by MCU structure. Requires `MCU_membership[]` map.
- **Non-Interleaved**: Single component per scan, MCU width/height = 1. Simpler downstream processing.

**Transcoder Architecture**: The `DCT_scaled_size = DCTSIZE` initialization hints at a design where JPEG→JPEG re-encoding tools skip full decompression. They read markers, understand the structure, and re-encode without inverse DCT/color conversion—useful for lossless repacking or requantization.

**Quantization Table Semantics in Progressive JPEG**: The `latch_quant_tables()` design is sophisticated. It preserves per-component quantization state across scans, supporting (rare but valid) cases where the encoder changes table definitions between scans of different components. Modern decoders often assume monotonic table definitions; this code is defensive.

**Non-Local Error Handling (ERREXIT)**: Unlike modern exception-based error handling, libjpeg uses `setjmp`/`longjmp` via ERREXIT macros. Errors unwind immediately to a handler in `jdapi.c`, bypassing normal function returns. This is why memory must be allocated **before** error-checking code: allocations won't be freed on ERREXIT.

**Idiomatic Q3A / libjpeg Patterns**:
- `LOCAL` / `METHODDEF` / `GLOBAL` visibility macros (K&R era compatibility)
- `JPP((...))` for function prototypes supporting both ANSI and K&R C
- Heavily abbreviated names (`comps_in_scan`, `h_samp_factor`, `iMCU`, `coef_bits`)
- Struct extension pattern: `my_input_controller` wraps `jpeg_input_controller` with private `inheaders` flag

## Potential Issues

1. **ERREXIT without cleanup**: If `initial_setup()` detects an error (e.g., image too big) and calls `ERREXIT`, any allocated state is abandoned. This is acceptable for error paths (decompression fails), but worth noting for robustness.

2. **Implicit dependency on marker reader state**: `consume_markers()` relies on `cinfo->marker->saw_SOF` being set by jdmarker. If marker reader is buggy or corrupted input is fed, this check can miss tables-only streams. However, the ERREXIT handles this.

3. **Bounds check on MCU_membership**: The code correctly checks `if (cinfo->blocks_in_MCU + mcublks > D_MAX_BLOCKS_IN_MCU)` before array writes, but this assumes the interleaved scan limit is reasonable. Defensive but not against pathological input with extreme sampling factors.

4. **No validation of sampling factor combinations**: The code validates `h_samp_factor > 0 && h_samp_factor <= MAX_SAMP_FACTOR`, but doesn't cross-check for nonsensical combinations (e.g., Cb/Cr at higher resolution than Y, which is unusual). jdmarker likely enforces JPEG spec, so this may be acceptable.

5. **Multi-scan loop prevention**: The `output_scan_number` clamping on EOI prevents infinite loops if the user requests more scans than exist. However, this is a band-aid; the real issue is user error (requesting too many scans). Clear documentation is important.

---

**Connection to Q3A Architecture**: This file is a **leaf module** in the renderer subsystem, with no cross-cutting dependencies to game logic, physics, or networking. It participates solely in the texture-loading pipeline (`tr_image.c` → libjpeg → GPU). The vendored IJG code is architecturally isolated and could be swapped for a different JPEG library with minimal changes to the renderer's public interface.
