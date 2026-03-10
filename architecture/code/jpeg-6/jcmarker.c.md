# code/jpeg-6/jcmarker.c

## File Purpose
Implements the JPEG marker writer module for the IJG JPEG compression library. It serializes all required JPEG datastream markers (SOI, SOF, SOS, DHT, DQT, DRI, APP0, APP14, EOI, etc.) to the output destination buffer.

## Core Responsibilities
- Emit raw bytes and 2-byte big-endian integers to the output destination
- Write quantization table markers (DQT) and Huffman table markers (DHT)
- Write frame header (SOFn) and scan header (SOS, DRI) markers
- Write file header (SOI + optional JFIF APP0 / Adobe APP14) and trailer (EOI)
- Write abbreviated table-only datastreams
- Initialize the `jpeg_marker_writer` vtable on `cinfo->marker`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `JPEG_MARKER` | enum | Symbolic names for all JPEG marker byte codes (0x01–0x100) |

## Global / File-Static State
None.

## Key Functions / Methods

### emit_byte
- **Signature:** `LOCAL void emit_byte(j_compress_ptr cinfo, int val)`
- **Purpose:** Writes one byte to the destination buffer; flushes the buffer if full.
- **Inputs:** `cinfo` — compress context; `val` — byte value to write
- **Outputs/Return:** void
- **Side effects:** Advances `dest->next_output_byte`, decrements `dest->free_in_buffer`; may call `empty_output_buffer` callback
- **Calls:** `dest->empty_output_buffer`, `ERREXIT`
- **Notes:** Suspension is not supported — if `empty_output_buffer` returns FALSE, it triggers a fatal error.

### emit_dqt
- **Signature:** `LOCAL int emit_dqt(j_compress_ptr cinfo, int index)`
- **Purpose:** Emits a DQT (quantization table) marker; suppresses duplicate emission via `sent_table` flag.
- **Inputs:** `index` — quant table slot (0–3)
- **Outputs/Return:** precision (0 = 8-bit, 1 = 16-bit values present)
- **Side effects:** Sets `qtbl->sent_table = TRUE`; writes to output
- **Calls:** `emit_marker`, `emit_2bytes`, `emit_byte`, `ERREXIT1`

### emit_dht
- **Signature:** `LOCAL void emit_dht(j_compress_ptr cinfo, int index, boolean is_ac)`
- **Purpose:** Emits a DHT (Huffman table) marker for DC or AC table; suppresses duplicates.
- **Side effects:** Sets `htbl->sent_table = TRUE`; writes to output
- **Calls:** `emit_marker`, `emit_2bytes`, `emit_byte`, `ERREXIT1`

### emit_sof
- **Signature:** `LOCAL void emit_sof(j_compress_ptr cinfo, JPEG_MARKER code)`
- **Purpose:** Emits a SOFn start-of-frame marker encoding image dimensions, precision, and per-component parameters.
- **Calls:** `emit_marker`, `emit_2bytes`, `emit_byte`, `ERREXIT1`

### emit_sos
- **Signature:** `LOCAL void emit_sos(j_compress_ptr cinfo)`
- **Purpose:** Emits a SOS (start-of-scan) marker; handles progressive mode table selection logic.
- **Calls:** `emit_marker`, `emit_2bytes`, `emit_byte`

### write_file_header
- **Signature:** `METHODDEF void write_file_header(j_compress_ptr cinfo)`
- **Purpose:** Writes SOI + optional JFIF APP0 and/or Adobe APP14 markers.
- **Calls:** `emit_marker`, `emit_jfif_app0`, `emit_adobe_app14`

### write_frame_header
- **Signature:** `METHODDEF void write_frame_header(j_compress_ptr cinfo)`
- **Purpose:** Emits DQT tables for all components then selects and emits the appropriate SOFn marker (baseline, extended, progressive, or arithmetic).
- **Calls:** `emit_dqt`, `emit_sof`, `TRACEMS`

### write_scan_header
- **Signature:** `METHODDEF void write_scan_header(j_compress_ptr cinfo)`
- **Purpose:** Emits DHT (or DAC for arithmetic) tables needed for the current scan, optional DRI, then SOS.
- **Calls:** `emit_dac`, `emit_dht`, `emit_dri`, `emit_sos`

### write_tables_only
- **Signature:** `METHODDEF void write_tables_only(j_compress_ptr cinfo)`
- **Purpose:** Writes an abbreviated JPEG containing only SOI + DQT + DHT + EOI (no image data).
- **Calls:** `emit_marker`, `emit_dqt`, `emit_dht`

### jinit_marker_writer
- **Signature:** `GLOBAL void jinit_marker_writer(j_compress_ptr cinfo)`
- **Purpose:** Allocates the `jpeg_marker_writer` subobject and populates its method pointers. Engine entry point called during compressor initialization.
- **Side effects:** Allocates from `JPOOL_IMAGE`; writes `cinfo->marker`
- **Calls:** `cinfo->mem->alloc_small`

### Notes
- `emit_marker`, `emit_2bytes` are trivial 1–2 line wrappers around `emit_byte`.
- `emit_dac`, `emit_dri`, `emit_jfif_app0`, `emit_adobe_app14`, `write_any_marker`, `write_file_trailer` are straightforward serializers not detailed above.

## Control Flow Notes
`jinit_marker_writer` is called at compressor init time. The compression pipeline then calls the installed method pointers in order: `write_file_header` → `write_frame_header` → (per scan) `write_scan_header` → [compressed data] → `write_file_trailer`. `write_tables_only` is an alternative path for abbreviated datastreams.

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_marker_writer`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `TRACEMS` macros (defined elsewhere via error manager)
- `C_ARITH_CODING_SUPPORTED` — conditional compile guard for `emit_dac` body (defined in `jconfig.h`)
