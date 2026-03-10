# code/jpeg-6/jdmarker.c

## File Purpose
Implements JPEG datastream marker parsing for the IJG decompressor. It reads and decodes all standard JPEG markers (SOI, SOF, SOS, DHT, DQT, DRI, DAC, APP0, APP14, etc.) with full support for input suspension—if insufficient data is available, parsing aborts and resumes transparently on the next call.

## Core Responsibilities
- Scan the input stream for JPEG marker bytes (0xFF prefix sequences)
- Parse each marker's parameter segment and populate `j_decompress_ptr` fields
- Support suspendable I/O: return `FALSE` mid-parse if data runs out; resume on re-entry
- Install and dispatch per-marker handler function pointers (APPn, COM)
- Implement restart-marker synchronization and error recovery (`jpeg_resync_to_restart`)
- Initialize the `jpeg_marker_reader` subobject at decompressor creation time

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `JPEG_MARKER` | enum | Symbolic names for all JPEG marker byte codes (0x01–0x100) |

## Global / File-Static State
None.

## Key Functions / Methods

### get_soi
- **Signature:** `LOCAL boolean get_soi(j_decompress_ptr cinfo)`
- **Purpose:** Process SOI (Start of Image) marker; reset per-image state.
- **Inputs:** `cinfo` decompressor context.
- **Outputs/Return:** `TRUE` always (no parameter bytes to read).
- **Side effects:** Resets arithmetic coding tables, `restart_interval`, colorspace flags, JFIF/Adobe flags; sets `marker->saw_SOI`.
- **Calls:** `TRACEMS`, `ERREXIT`.
- **Notes:** Errors on duplicate SOI.

### get_sof
- **Signature:** `LOCAL boolean get_sof(j_decompress_ptr cinfo, boolean is_prog, boolean is_arith)`
- **Purpose:** Parse SOFn marker; populate image dimensions, component descriptors.
- **Inputs:** `is_prog`/`is_arith` encode which SOF variant was found.
- **Outputs/Return:** `FALSE` if suspended; `TRUE` on success.
- **Side effects:** Allocates `cinfo->comp_info` array (JPOOL_IMAGE); sets `image_width/height`, `num_components`, per-component sampling factors and quantization table indices; sets `marker->saw_SOF`.
- **Calls:** `INPUT_2BYTES`, `INPUT_BYTE`, `INPUT_SYNC`, `alloc_small`, `ERREXIT`, `TRACEMS4`.

### get_sos
- **Signature:** `LOCAL boolean get_sos(j_decompress_ptr cinfo)`
- **Purpose:** Parse SOS (Start of Scan) marker; bind component table selectors for the current scan.
- **Inputs:** `cinfo`.
- **Outputs/Return:** `FALSE` if suspended; `TRUE` on success.
- **Side effects:** Fills `cinfo->cur_comp_info[]`, `comps_in_scan`, progressive params (`Ss`, `Se`, `Ah`, `Al`); increments `input_scan_number`; resets `next_restart_num`.
- **Calls:** `INPUT_2BYTES`, `INPUT_BYTE`, `INPUT_SYNC`, `ERREXIT`, `TRACEMS`.

### get_app0
- **Signature:** `METHODDEF boolean get_app0(j_decompress_ptr cinfo)`
- **Purpose:** Parse APP0; detect and extract JFIF header (version, pixel density, thumbnail dimensions).
- **Side effects:** Sets `cinfo->saw_JFIF_marker`, `density_unit`, `X_density`, `Y_density`. Skips remaining bytes via `skip_input_data`.
- **Calls:** `INPUT_2BYTES`, `INPUT_BYTE`, `INPUT_SYNC`, `skip_input_data`, `WARNMS2`, `TRACEMS`.

### get_app14
- **Signature:** `METHODDEF boolean get_app14(j_decompress_ptr cinfo)`
- **Purpose:** Parse Adobe APP14 marker; extract color transform code.
- **Side effects:** Sets `cinfo->saw_Adobe_marker`, `Adobe_transform`.
- **Calls:** `INPUT_2BYTES`, `INPUT_BYTE`, `INPUT_SYNC`, `skip_input_data`.

### get_dht
- **Signature:** `LOCAL boolean get_dht(j_decompress_ptr cinfo)`
- **Purpose:** Parse DHT marker; allocate and populate Huffman table entries.
- **Side effects:** Allocates `JHUFF_TBL` objects; fills `dc_huff_tbl_ptrs` / `ac_huff_tbl_ptrs`.
- **Calls:** `jpeg_alloc_huff_table`, `MEMCOPY`, `INPUT_SYNC`, `ERREXIT`.

### get_dqt
- **Signature:** `LOCAL boolean get_dqt(j_decompress_ptr cinfo)`
- **Purpose:** Parse DQT marker; fill quantization table arrays (8- or 16-bit precision).
- **Side effects:** Allocates `JQUANT_TBL`; fills `cinfo->quant_tbl_ptrs[]`.
- **Calls:** `jpeg_alloc_quant_table`, `INPUT_SYNC`, `ERREXIT`.

### get_dri
- **Signature:** `LOCAL boolean get_dri(j_decompress_ptr cinfo)`
- **Purpose:** Parse DRI marker; set restart interval.
- **Side effects:** Sets `cinfo->restart_interval`.

### get_dac
- **Signature:** `LOCAL boolean get_dac(j_decompress_ptr cinfo)`
- **Purpose:** Parse DAC marker; populate arithmetic-coding conditioning tables.
- **Side effects:** Writes `cinfo->arith_dc_L/U[]`, `arith_ac_K[]`.

### next_marker
- **Signature:** `LOCAL boolean next_marker(j_decompress_ptr cinfo)`
- **Purpose:** Scan forward in the byte stream to the next valid JPEG marker (0xFF + non-zero, non-FF byte).
- **Outputs/Return:** `FALSE` on suspension; sets `cinfo->unread_marker` on success.
- **Side effects:** Increments `marker->discarded_bytes` for any garbage bytes encountered; warns if nonzero.

### first_marker
- **Signature:** `LOCAL boolean first_marker(j_decompress_ptr cinfo)`
- **Purpose:** Strict initial marker read: demands the very first two bytes are `0xFF 0xD8` (SOI); no garbage tolerance.
- **Notes:** Prevents scanning an entire non-JPEG file looking for SOI.

### read_markers
- **Signature:** `METHODDEF int read_markers(j_decompress_ptr cinfo)`
- **Purpose:** Main marker-dispatch loop; called by the input controller to consume markers until SOS or EOI.
- **Outputs/Return:** `JPEG_SUSPENDED`, `JPEG_REACHED_SOS`, or `JPEG_REACHED_EOI`.
- **Side effects:** Dispatches to all per-marker handlers; uses `marker->process_APPn[]` and `marker->process_COM` function-pointer tables for extensible APP/COM handling.

### read_restart_marker
- **Signature:** `METHODDEF boolean read_restart_marker(j_decompress_ptr cinfo)`
- **Purpose:** Consume an expected RSTn marker between MCU groups; invokes `resync_to_restart` on mismatch.
- **Side effects:** Advances `marker->next_restart_num`; may clear `unread_marker`.

### jpeg_resync_to_restart
- **Signature:** `GLOBAL boolean jpeg_resync_to_restart(j_decompress_ptr cinfo, int desired)`
- **Purpose:** Default restart-error recovery: decides whether to discard the stray marker, scan forward for the next marker, or leave it unread for the entropy decoder to handle.
- **Notes:** Three-action strategy based on how far the found marker is from the expected one (±2 modulo 8 wrapping).

### jinit_marker_reader
- **Signature:** `GLOBAL void jinit_marker_reader(j_decompress_ptr cinfo)`
- **Purpose:** One-time initialization; allocates `jpeg_marker_reader` in permanent pool and installs all method pointers.
- **Side effects:** Installs `get_app0` for APP0, `get_app14` for APP14, `skip_variable` for all other APPn and COM; calls `reset_marker_reader`.

## Control Flow Notes
Called during JPEG decompression startup and per-scan input. `jinit_marker_reader` runs once at object creation. `read_markers` is the main entry point polled by `jdinput.c`'s input controller each time more data is available, returning status codes that gate the decompressor's state machine progression from header parsing → scan data → EOI.

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF` macros
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_marker_reader`, `JHUFF_TBL`, `JQUANT_TBL`, `jpeg_component_info`, all `JPEG_*` status codes
- `jerror.h` — `ERREXIT`, `WARNMS2`, `TRACEMS*` macros
- **Defined elsewhere:** `jpeg_alloc_huff_table`, `jpeg_alloc_quant_table` (jcomapi.c); `datasrc->fill_input_buffer`, `skip_input_data`, `resync_to_restart` (source manager, e.g. jdatasrc.c)
