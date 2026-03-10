# code/jpeg-6/jctrans.c — Enhanced Analysis

## Architectural Role

This file implements JPEG **transcoding-specific** compression infrastructure, a narrow but distinct use case within the broader engine. While the engine's renderer loads pre-compressed texture JPEGs at runtime (via `jload.c`), this transcode path exists to support offline tooling that re-encodes JPEG coefficient data without full pixel-space decompression—a lossless preservation of DCT precision. It does not participate in the runtime texture pipeline; instead, it's a utility for tools (map compilers, asset converters) that need to manipulate already-compressed JPEG streams.

## Key Cross-References

### Incoming (who depends on this file)
- **Tools & utilities only**: Offline compilers and asset processing tools that perform `jpeg_read_coefficients` → `jpeg_copy_critical_parameters` + `jpeg_write_coefficients` → `jpeg_finish_compress` workflows. The runtime engine (client/server/renderer) **never calls** these transcoding entry points.
- No references from `code/renderer`, `code/client`, or `code/server` game code paths.

### Outgoing (what this file depends on)
- **JPEG infrastructure** (`jpeglib.h`, `jinclude.h`): All type definitions, error codes, memory/method callback structs.
- **Other libjpeg-6 modules** (all called via indirect function pointers in `j_compress_ptr cinfo`):
  - `jpeg_suppress_tables`, `jpeg_set_defaults`, `jpeg_set_colorspace`, `jpeg_alloc_quant_table` (parameter copying)
  - `jinit_c_master_control` (compression init)
  - `jinit_huff_encoder`, `jinit_phuff_encoder` (entropy coding mode selection)
  - `jinit_marker_writer` (file structure output)
  - `jzero_far` (memory zeroing utility)
  - Virtual array access/realization via `cinfo->mem->*` function pointers

## Design Patterns & Rationale

**Virtual Arrays Pattern**: The JPEG library abstracts all large buffers (DCT coefficient blocks) behind a virtual-array interface, allowing transparent disk-swapping if RAM is insufficient. The transcode path reads from pre-supplied virtual arrays rather than generating them from pixel data—enabling zero-copy coefficient passing.

**Method Table / VTable Pattern**: The `my_coef_controller` struct extends JPEG's base `jpeg_c_coef_controller` with method pointers (`start_pass_coef`, `compress_data`). This allows the transcode module to plug a custom coefficient source into the standard compression pipeline without modifying core JPEG code.

**Stateful MCU Iteration**: The `my_coef_controller` tracks iMCU row, MCU column, and vertical offsets within an iMCU row, enabling progressive output of MCU data across multiple calls to `compress_output`. This permits the entropy encoder to suspend if output buffers fill, then resume on the next pump cycle—a suspension/resumption pattern inherited from normal compression.

**On-the-Fly Dummy Padding**: Rather than requiring input virtual arrays to contain dummy blocks at image right/bottom boundaries (which would waste space), this code generates them during output. Dummy blocks are initialized with zero AC coefficients and inherit the DC value from the preceding real block—a detail matching `jccoefct.c`'s normal compression strategy.

## Data Flow Through This File

1. **Setup Phase** (`jpeg_write_coefficients`):
   - Caller has already set compress parameters and provided pre-allocated virtual arrays.
   - File resets error/destination infrastructure, calls `transencode_master_selection` to wire compression modules, and writes SOI (Start of Image) marker.

2. **Coefficient Extraction** (during `jpeg_finish_compress` iteration):
   - `compress_output` is called repeatedly, once per iMCU row.
   - It accesses the virtual arrays using the memory manager's `access_virt_barray` callback.
   - For each MCU, it constructs a list of DCT block pointers from the arrays.
   - Dummy blocks (at right/bottom edges) are injected from pre-allocated, zero-initialized buffer.

3. **Entropy Encoding**:
   - MCU block list is passed to `cinfo->entropy->encode_mcu`, which performs Huffman/arithmetic coding.
   - If the encoder suspends (output buffer full), state counters are saved for resumption on next call.

4. **File Output**:
   - Frame/scan headers and entropy-encoded data flow to the destination (`cinfo->dest`) callback—ultimately written to disk or memory.

## Learning Notes

**Idiomatic JPEG Library Pattern**: This file demonstrates the standard JPEG library's extensibility model circa 1995. Rather than exposing a monolithic "do everything" API, the library defines abstract interfaces (compress object with pluggable modules) that tools can customize. This transcode module is a textbook example of how a library consumer can replace the pixel-input stage with a coefficient-input stage.

**Historical Context**: JPEG transcoding (re-encoding without pixel decoding) was important in the mid-1990s when disk and memory were constrained and lossless JPEG transformations were needed (e.g., rotating images without DCT recomputation). Modern engines rarely use this pattern; GPUs handle re-encoding. Quake III likely included it for offline asset preprocessing rather than runtime use.

**MCU as Atomic Unit**: The code is written around JPEG's MCU (Minimum Coded Unit)—the smallest independently-coded rectangle, typically 8×8 or 16×16 pixels depending on chroma subsampling. Modern developers might expect block-by-block or line-by-line iteration; JPEG insists on MCU-granularity for entropy coding, hence the `iMCU_row_num` / `MCU_col_num` / `MCU_vert_offset` triplet.

**No ECS / Scene Graphs**: This is pure codec-level machinery with no entity, component, or render-graph abstraction. It's procedural, imperative JPEG library integration—representative of middleware C libraries from that era.

## Potential Issues

- **Arithmetic Coding Unsupported**: `JERR_ARITH_NOTIMPL` is hard-coded; if an input image uses arithmetic coding, transcode will fail. This is a library limitation (arithmetic coding was patented; IJG excluded it). Modern JPEG2000/HEIF tools would handle this differently.

- **No Runtime Transcode Path in Engine**: The game engine itself never calls these functions. If a designer or tool developer mistakenly tries to use transcode during gameplay (e.g., dynamic texture re-encoding), it will fail silently or cause state corruption. No defensive guards prevent misuse in game code.

- **Quantization Table Validation Strict**: `jpeg_copy_critical_parameters` validates that each component's assigned quantization table matches the actual slot contents. If a source image reused a quantization table slot for multiple logical tables (valid in JPEG, but non-standard), the transcode will error with `JERR_MISMATCHED_QUANT_TABLE`. This conservative validation is safer but less flexible than full table cloning.
