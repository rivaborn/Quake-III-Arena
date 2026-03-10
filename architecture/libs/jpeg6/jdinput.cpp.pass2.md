# libs/jpeg6/jdinput.cpp — Enhanced Analysis

## Architectural Role

This file implements JPEG decompression input control within the vendored libjpeg-6 library, which the engine integrates solely for **texture asset loading** in the renderer (`code/renderer/tr_image.c` → `jload.c`). It manages the parser state machine that sequentially reads JPEG markers, validates image metadata, and coordinates handoff between the marker-reading and coefficient-decoding phases. As part of a self-contained external library, it has **no direct cross-subsystem dependencies** beyond the error/memory infrastructure provided by libjpeg's own initialization layer.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture loading** (`code/renderer/tr_image.c`): Indirectly via the libjpeg public API (`jpeg_start_decompress` → calls into this module)
- **IJG libjpeg flow**: `jdmaster.c` calls `start_input_pass()` after decompressor setup; coefficient controller calls `finish_input_pass()` after data consumption
- No engine-subsystem direct calls; entirely hidden behind libjpeg's public `jpeg_decompress_struct` interface

### Outgoing (what this file depends on)
- **libjpeg infrastructure**: Error macros (`ERREXIT`, `ERREXIT1`, `ERREXIT2`), memory allocator (`cinfo->mem->alloc_small`), marker reader (`cinfo->marker->read_markers`), entropy decoder (`cinfo->entropy->start_pass`)
- **libjpeg types**: `j_decompress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, etc.
- No engine dependencies; entirely self-contained within vendored JPEG library

## Design Patterns & Rationale

- **State Machine** (`my_input_controller.inheaders`): Tracks whether parser is still in header phase (pre-SOS) or actively decoding scans. Enforces single-entry invariant: `initial_setup()` must be called exactly once at first SOS, preventing re-initialization bugs.

- **Method Pointer Dispatch** (`inputctl->pub.consume_input`, `cinfo->entropy->start_pass`): Callback-driven architecture allows different entropy encoders (Huffman, progressive, etc.) to inject custom behavior without tight coupling. The pointer swaps between `consume_markers` (inter-segment) and `coef->consume_data` (during scan) to implement the two-phase parser.

- **Lazy Quantization Table Binding** (`latch_quant_tables`): Defers Q-table copying until first use within a scan, supporting multi-scan JPEG files where the same Q-table slot number may hold different table definitions in different scans. Matches JPEG spec constraint that a component's Q-table slot contents are immutable within a single component scan.

- **Interleaving vs. Non-Interleaving Branching** (`per_scan_setup`): Single-component scans use simplified MCU layout (one block per MCU, no multiplexing); multi-component scans compute combined MCU structure. Reflects fundamental JPEG compression difference but unified under same component array.

## Data Flow Through This File

1. **Initialization** (`jinit_input_controller`): Allocates `my_input_controller` state; installs method pointers; sets `inheaders=TRUE`.

2. **Header Phase** (`consume_markers` loops): Reads markers via engine-provided reader; at first SOS:
   - `initial_setup()`: Validates image dimensions, precision, component count; computes max sampling factors; derives block/MCU dimensions
   - Sets `inheaders=FALSE`; awaits `start_input_pass()` call from `jdmaster.c`

3. **Per-Scan Setup** (each SOS): `start_input_pass()` dispatches to:
   - `per_scan_setup()`: Computes scan-specific MCU geometry (interleaving vs. single-component); populates `MCU_membership[]` array for coefficient block routing
   - `latch_quant_tables()`: Snapshots current Q-table pointers into component state for later dequantization
   - `cinfo->entropy->start_pass()` + `cinfo->coef->start_input_pass()`: Activates entropy decoder and coefficient handler
   - Swaps `consume_input` callback from `consume_markers` to `coef->consume_data`

4. **Scan Data Phase**: Coefficient controller owns input; `finish_input_pass()` re-installs `consume_markers` for next inter-segment phase.

5. **EOI Handling**: `consume_markers` detects end-of-image; validates no orphaned output scans; sets `eoi_reached` flag.

## Learning Notes

- **Marker-Driven State Machine**: Entire JPEG parsing flow is event-driven by marker discovery (SOS, EOI, DQT, DHT, etc.), not imperative loops. Reflects 1990s codec design where streams arrive incrementally (e.g., from network or file chunks). Modern engines typically parse full file header upfront.

- **Progressive JPEG Support**: The `per_scan_setup()` branching on `cinfo->comps_in_scan` accommodates progressive encoding (multiple partial-spectrum scans per component), but the core engine uses this for baseline/sequential only in practice.

- **Q-Table Lifecycle Separation**: Copying quantization tables at first-scan-use (not at marker read time) is a subtle design choice reflecting the JPEG spec's permission for slot-sharing across multiple components with deferred table definition. Shows defensive parsing typical of 1990s codec libraries.

- **Idiomatic IJG Patterns**: Heavy use of macro-based error handling (`ERREXIT`), factory methods (`jinit_*`), struct-embedded public interface (`pub` member), and typedefs for opaque pointers (`my_inputctl_ptr`). Pre-dates C99 and modern practices; no inline structs or const correctness.

## Potential Issues

- **No Bounds Checking on `cinfo->comps_in_scan`**: The validation `if (cinfo->comps_in_scan <= 0 || cinfo->comps_in_scan > MAX_COMPS_IN_SCAN)` occurs *after* the non-interleaved branch uses `cinfo->cur_comp_info[0]` directly. A malformed JPEG with `comps_in_scan == 0` at non-interleaved SOS would dereference null; the error path in the interleaved branch is unreachable for this case. (Likely harmless in practice: most malformed JPEGs fail at marker validation before reaching this point.)

- **No Validation of Component Sampling Factors**: `initial_setup()` checks `h_samp_factor` and `v_samp_factor` are in range, but doesn't verify they are non-zero or consistent with MCU block counts. If a component claims `h_samp_factor > max_h_samp_factor` post-validation (encoding violation), `per_scan_setup` would compute nonsensical MCU membership arrays.
