# code/jpeg-6/jdmarker.c — Enhanced Analysis

## Architectural Role

This file is the **JPEG marker parser** layer within the vendored IJG libjpeg-6 library, integrated into the rendering pipeline for runtime texture decompression. The renderer (`code/renderer/tr_image.c`) invokes `jload.c` when a JPEG texture is encountered, which in turn uses this marker reader to parse the JPEG stream's structural metadata (dimensions, color space, Huffman/quantization tables). By supporting input suspension, jdmarker.c enables non-blocking texture loading: if the source manager detects incomplete data, parsing yields control back to the engine without blocking the frame loop, resuming transparently when more bytes arrive.

## Key Cross-References

### Incoming (who depends on this file)
- **jdinput.c** (IJG internal): The input controller loop that repeatedly calls `read_markers()` and dispatches to appropriate frame-processing stages based on return status (`JPEG_SUSPENDED`, `JPEG_REACHED_SOS`, `JPEG_REACHED_EOI`).
- **jdapimin.c** / **jdapistd.c** (IJG decompressor init): Calls `jinit_marker_reader()` exactly once per decompressor instance to initialize the `jpeg_marker_reader` state machine and install all method pointers.
- **Indirectly from code/renderer/tr_image.c**: When `R_LoadJPG()` is called during `R_RegisterImage()`, the renderer passes a data source to libjpeg, which eventually reaches jdmarker.c via the standard JPEG decompression pipeline.

### Outgoing (what this file depends on)
- **jpeglib.h** / **jpegint.h** types: `j_decompress_ptr`, `jpeg_marker_reader`, `jpeg_component_info`, `JHUFF_TBL`, `JQUANT_TBL`, all status codes (`JPEG_SUSPENDED`, etc.).
- **jerror.h** diagnostic macros: `ERREXIT`, `WARNMS`, `TRACEMS*` — funneled through cinfo's error handler (e.g., `cinfo->err->error_exit`, defaulting to `jpeg_std_error`).
- **jcomapi.c** factory functions: `jpeg_alloc_huff_table()`, `jpeg_alloc_quant_table()` — allocate table objects from the permanent memory pool.
- **Data source manager** (`cinfo->src`): Function pointers `fill_input_buffer()`, `skip_input_data()`, and fields `next_input_byte`, `bytes_in_buffer`. These are provided by the caller (e.g., `jdatasrc.c` for file-based I/O or a custom source for in-memory decompression).
- **Memory manager** (`cinfo->mem`): `alloc_small()` for component info allocation; uses `JPOOL_IMAGE` lifetime.

## Design Patterns & Rationale

1. **Suspendable I/O via macro-based buffer abstraction** (INPUT_BYTE, INPUT_2BYTES, INPUT_SYNC, INPUT_RELOAD):
   - Each marker handler maintains local copies of `datasrc->next_input_byte` and `bytes_in_buffer` for performance (avoiding pointer dereferences in hot loops).
   - When data runs out, `MAKE_BYTE_AVAIL` calls `fill_input_buffer()` (callback to source manager) and reloads locals. If fill returns FALSE (no more data), the action (typically `return FALSE`) unwinds the call stack.
   - On re-entry (after source provides more bytes), `read_markers()` re-invokes the same marker handler, which starts fresh—no continuation state needed because INPUT_VARS reloads the source manager's state.
   - **Rationale**: This pattern avoids explicit state machines per marker. Suspendable design allows texture loading to interleave with frame rendering; no buffer-all-JPEG-data requirement.

2. **Per-marker handler dispatch via function pointers** (`marker->process_APPn[]`, `marker->process_COM`):
   - Standard handlers are built-in (`get_app0`, `get_app14`); others default to `skip_variable`.
   - Allows extensibility without modifying core jdmarker.c (though in practice, built-in handlers cover all JPEG use cases).
   - **Rationale**: JPEG is extensible via APPn markers. This design accommodates proprietary metadata (e.g., camera EXIF in APP1, Canon extensions in proprietary vendors' markers) without forcing the parser to understand them.

3. **Lazy allocation of component array** (get_sof):
   - `cinfo->comp_info` is allocated on first SOF encounter, guarded by `if (cinfo->comp_info == NULL)`.
   - Even if SOF is interrupted (suspension), subsequent re-entry skips re-allocation.
   - **Rationale**: JPEG must know component count before allocating the descriptor array. Lazy allocation avoids dummy sizing; once SOF arrives, the allocation is final for this image.

4. **Restart marker synchronization** (read_restart_marker, jpeg_resync_to_restart):
   - Embedded RSTn markers (every N MCUs) allow error recovery: if a byte error corrupts scan data, the decoder can resynchronize at the next restart marker.
   - `jpeg_resync_to_restart()` implements a lenient three-way decision: advance, stay, or skip to next restart based on distance from expected marker number (modulo 8, wrapping ±2 tolerance).
   - **Rationale**: JPEG's error resilience; scan data loss can be tolerated by jumping to next safe synchronization point.

5. **Marker re-reading pattern** (`cinfo->unread_marker`):
   - When a marker is encountered out of expected sequence, it's stored in `unread_marker` rather than discarded.
   - Subsequent `read_markers()` entry checks `unread_marker != 0` and re-dispatches it.
   - **Rationale**: No lookahead buffering; single-byte pushback via state variable is simpler than ring buffers.

## Data Flow Through This File

```
Input Stream (JPEG bytes)
    ↓
[first_marker / next_marker]: scan for 0xFF XX (marker code)
    ↓ (marker byte cached in cinfo->unread_marker)
read_markers(): dispatch loop
    ├─→ SOI: reset state, verify "first marker is SOI"
    ├─→ SOF0-SOF15: parse dimensions, allocate comp_info[], store sampling factors
    ├─→ DHT: allocate Huffman table, read bits[], huffval[]
    ├─→ DQT: allocate quantization table, read precision + 64/256 Q values
    ├─→ DRI: store restart interval
    ├─→ DAC: populate arithmetic coding tables (rarely used in practice)
    ├─→ APP0: detect JFIF marker, extract density, thumbnail dims
    ├─→ APP14: detect Adobe marker, extract color transform
    ├─→ SOS: bind current scan's component selectors, store Ss/Se/Ah/Al (progressive params)
    └─→ EOI: signal decompression complete
    ↓
[jdinput.c input controller returns status]
    ├─ JPEG_REACHED_SOS → start entropy decoding (scan data phase)
    ├─ JPEG_SUSPENDED → wait for more input bytes, re-call read_markers()
    └─ JPEG_REACHED_EOI → finish decompression

State accumulated in j_decompress_ptr:
  - cinfo->image_width, image_height, num_components
  - cinfo->comp_info[]: per-component h_samp_factor, v_samp_factor, quant_tbl_no
  - cinfo->dc_huff_tbl_ptrs[], ac_huff_tbl_ptrs[]: Huffman tables
  - cinfo->quant_tbl_ptrs[]: quantization tables
  - cinfo->saw_JFIF_marker, X_density, Y_density, density_unit
  - cinfo->restart_interval, arith_dc_L/U[], arith_ac_K[]
  - cinfo->input_scan_number, comps_in_scan, Ss, Se, Ah, Al
```

## Learning Notes

**Historical Context (1991–1995 IJG design):**
- This is pre-C99, pre-OOP C code. Memory management is manual (`alloc_small`, no automatic cleanup). Error handling uses `setjmp/longjmp` (see `jerror.h`'s `ERREXIT` macro).
- The INPUT_*BYTE macros are **macro metaprogramming** before modern C preprocessor became standard. They embed control flow (`return FALSE` actions) within macro expansions—a pattern that would be controversial today but was idiomatic in embedded systems.
- The suspendable design is **forward-thinking for its era**: most decoders of the 1990s required all input data upfront (no streaming). IJG's approach allows incremental I/O, valuable for slow networks or real-time constraints.

**Modern Equivalents:**
- A modern JPEG decoder might use coroutines/async-await (if the language supports it), or explicit state machines with callbacks.
- The function-pointer dispatch pattern is superseded by OOP polymorphism in C++ or interface-based design in modern C.
- The macro abstraction is now handled by language features (e.g., Rust's `?` operator for error propagation).

**JPEG-Specific Insights:**
- **Markers are structural, not payload**: They describe image geometry, color space, and compression tables. The actual compressed data is in "scan" segments between SOS and RSTn/SOS/EOI markers.
- **Progressive JPEG**: SOF2 (vs. SOF0) signals progressive (multi-pass) encoding. The `Ss`, `Se`, `Ah`, `Al` parameters in SOS control which frequency bands are decoded in each scan.
- **Restart markers (RSTn)**: Embedded every N MCUs. They allow error recovery: if scan data is corrupted, resync at next RSTn. The `next_restart_num` counter detects out-of-sequence RSTn and triggers `jpeg_resync_to_restart()`.

**Why This File Matters in Q3:**
- Textures are compressed as JPEG on disk to reduce .pk3 file size.
- At load time, jdmarker.c parses the JPEG header to determine texture dimensions and colorspace **before** allocating GPU memory. This avoids allocating oversized buffers.
- The suspendable design means texture loading doesn't stall the frame loop if the .pk3 archive's I/O is slow.

## Potential Issues

1. **Restart Logic Edge Case**: The modulo-8 wrapping tolerance (±2) in `jpeg_resync_to_restart()` can mask true bit-error corruption if it happens to fall within the tolerance band. Modern decoders often verify checksum-like metadata (e.g., marker segment length fields) to catch corruption more reliably. IJG's approach is pragmatic for 1990s network conditions (drop frames rather than stall) but fragile for memory-mapped I/O.

2. **No Length Validation on DQT/DHT**: The code trusts that marker length fields are correct. A malformed JPEG with an invalid length could cause buffer overrun in `get_dqt()` or `get_dht()` if the loop reads past the allocated `bits[]` array. Bounds checking is implicit (via local buffer size) but not explicit.

3. **Mutable Dispatch Tables**: `marker->process_APPn[]` pointers are installed at init and never validated. If a memory corruption elsewhere in the engine overwrites these function pointers, jdmarker.c would call arbitrary code. Mitigation: the VM sandbox (j_decompress_ptr is allocated in engine-managed memory), but this is a general memory-safety concern, not specific to jdmarker.c.

4. **Implicit EOF Detection**: If the JPEG stream ends prematurely (e.g., truncated .pk3 extraction), `next_marker()` will eventually return `M_ERROR` (0x100, not a valid JPEG marker). However, the code doesn't explicitly validate `cinfo->unread_marker` against `M_ERROR` before dispatching in `read_markers()`. The error is caught downstream (in `read_scan_header()` or entropy decoder init), but earlier validation would be clearer.
