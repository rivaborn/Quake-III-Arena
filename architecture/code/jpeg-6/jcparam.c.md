# code/jpeg-6/jcparam.c

## File Purpose
Provides optional default-setting and parameter-configuration routines for the IJG JPEG compressor. Applications call these helpers to configure quantization tables, Huffman tables, colorspace, and encoding options before starting compression.

## Core Responsibilities
- Build and install scaled quantization tables from standard JPEG spec templates
- Convert user-friendly quality ratings (0–100) to quantization scale factors
- Install standard Huffman tables (DC/AC, luma/chroma) per JPEG spec section K.3
- Set all compressor defaults (quality 75, Huffman coding, no restart markers, etc.)
- Map input colorspace to JPEG output colorspace and configure per-component sampling
- Optionally generate a progressive JPEG scan script

## Key Types / Data Structures
None defined here; uses types from `jpeglib.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `JQUANT_TBL` | struct (external) | Holds 64 quantization values + `sent_table` flag |
| `JHUFF_TBL` | struct (external) | Holds bits/huffval arrays + `sent_table` flag |
| `jpeg_scan_info` | struct (external) | Describes one scan in a progressive script |
| `jpeg_component_info` | struct (external) | Per-component sampling factors and table selectors |
| `J_COLOR_SPACE` | enum (external) | Identifies color space (RGB, YCbCr, CMYK, etc.) |

## Global / File-Static State
None. All state is held in the caller-supplied `j_compress_ptr cinfo`.

## Key Functions / Methods

### jpeg_add_quant_table
- **Signature:** `GLOBAL void jpeg_add_quant_table(j_compress_ptr cinfo, int which_tbl, const unsigned int *basic_table, int scale_factor, boolean force_baseline)`
- **Purpose:** Allocates (if needed) and fills quantization table slot `which_tbl` by scaling `basic_table` entries by `scale_factor/100`.
- **Inputs:** `cinfo` — compressor state; `which_tbl` — table index 0–3; `basic_table` — 64-entry source array; `scale_factor` — percentage; `force_baseline` — clamp to 255.
- **Outputs/Return:** void; modifies `cinfo->quant_tbl_ptrs[which_tbl]` in place.
- **Side effects:** May allocate a `JQUANT_TBL` via `jpeg_alloc_quant_table`. Calls `ERREXIT1` if compression already started.
- **Calls:** `jpeg_alloc_quant_table`, `ERREXIT1`
- **Notes:** Values clamped to [1, 32767]; additionally to [1, 255] when `force_baseline` is TRUE. Sets `sent_table = FALSE` so table is written to the JPEG stream.

### jpeg_quality_scaling
- **Signature:** `GLOBAL int jpeg_quality_scaling(int quality)`
- **Purpose:** Maps a user quality value 0–100 to a quantization scale percentage using the IJG recommended curve.
- **Inputs:** `quality` — integer 0 (worst) to 100 (best).
- **Outputs/Return:** Scale factor percentage (e.g., 100 → quality 50; approaches 0 → quality 100).
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Quality < 50 maps to `5000/quality`; quality ≥ 50 maps to `200 - 2*quality`. Quality 0 is treated as 1.

### jpeg_set_quality
- **Signature:** `GLOBAL void jpeg_set_quality(j_compress_ptr cinfo, int quality, boolean force_baseline)`
- **Purpose:** High-level entry point: converts 0–100 quality to a scale factor and installs standard luma/chroma quant tables.
- **Calls:** `jpeg_quality_scaling`, `jpeg_set_linear_quality`

### jpeg_set_linear_quality
- **Signature:** `GLOBAL void jpeg_set_linear_quality(j_compress_ptr cinfo, int scale_factor, boolean force_baseline)`
- **Purpose:** Installs standard luminance and chrominance quantization tables at the given scale factor directly (no quality curve conversion).
- **Calls:** `jpeg_add_quant_table` (twice)

### jpeg_set_defaults
- **Signature:** `GLOBAL void jpeg_set_defaults(j_compress_ptr cinfo)`
- **Purpose:** Initializes all compressor parameters to reasonable defaults: quality 75, Huffman coding, no progressive/arithmetic/restart, JFIF pixel density 1:1, DCT method default.
- **Inputs:** `cinfo` — must have `in_color_space` set before call.
- **Side effects:** Allocates `comp_info` array in permanent pool; calls multiple setup helpers; errors if compression already started.
- **Calls:** `jpeg_set_quality`, `std_huff_tables`, `jpeg_default_colorspace`, `ERREXIT1`

### jpeg_default_colorspace
- **Signature:** `GLOBAL void jpeg_default_colorspace(j_compress_ptr cinfo)`
- **Purpose:** Selects the JPEG output colorspace based on `cinfo->in_color_space` (e.g., RGB → YCbCr).
- **Calls:** `jpeg_set_colorspace`, `ERREXIT`

### jpeg_set_colorspace
- **Signature:** `GLOBAL void jpeg_set_colorspace(j_compress_ptr cinfo, J_COLOR_SPACE colorspace)`
- **Purpose:** Sets `cinfo->jpeg_color_space`, `num_components`, and per-component IDs/sampling factors/table assignments. Also sets JFIF/Adobe marker flags.
- **Calls:** `ERREXIT1`, `ERREXIT2`, `ERREXIT`; uses `SET_COMP` macro.
- **Notes:** YCbCr defaults to 2×2 chroma subsampling. RGB and CMYK/YCCK write Adobe marker; Grayscale/YCbCr write JFIF marker.

### jpeg_simple_progression
- **Signature:** `GLOBAL void jpeg_simple_progression(j_compress_ptr cinfo)`
- **Purpose:** Allocates and populates a recommended progressive scan script in `cinfo->scan_info`.
- **Calls:** `fill_dc_scans`, `fill_a_scan`, `fill_scans`, `ERREXIT1`
- **Notes:** Only compiled when `C_PROGRESSIVE_SUPPORTED` is defined. YCbCr gets a custom 10-scan script; other spaces get a generic script sized as `2 + 4*ncomps` (or `6*ncomps` if components exceed `MAX_COMPS_IN_SCAN`).

### Notes on helpers
- `add_huff_table` (LOCAL): Allocates and copies bits/huffval into a `JHUFF_TBL` slot; sets `sent_table = FALSE`.
- `std_huff_tables` (LOCAL): Installs all four standard Huffman tables (DC/AC × luma/chroma) from JPEG spec K.3 static arrays.
- `fill_a_scan`, `fill_scans`, `fill_dc_scans` (LOCAL): Helpers for `jpeg_simple_progression` that populate `jpeg_scan_info` entries.

## Control Flow Notes
This file is a **pre-compression setup** module. All functions must be called before `jpeg_start_compress()` is invoked — enforced by `global_state != CSTATE_START` guards. Typical flow: application calls `jpeg_set_defaults()` → optionally overrides specific parameters → calls `jpeg_start_compress()`. This file has no role during the frame loop or shutdown.

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF`, platform warning suppression
- `jpeglib.h` / `jpegint.h` / `jerror.h` (via `JPEG_INTERNALS`) — all struct definitions, constants, error macros
- **Defined elsewhere:** `jpeg_alloc_quant_table`, `jpeg_alloc_huff_table` (memory module); `ERREXIT`/`ERREXIT1`/`ERREXIT2` (error handler macros); `CSTATE_START`, `JPOOL_PERMANENT`, `BITS_IN_JSAMPLE`, `DCTSIZE2`, `MAX_COMPONENTS`, `NUM_ARITH_TBLS` (constants from jpegint.h/jconfig.h)
