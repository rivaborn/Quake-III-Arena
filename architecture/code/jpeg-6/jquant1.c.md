# code/jpeg-6/jquant1.c

## File Purpose
Implements 1-pass color quantization for JPEG decompression, mapping full-color pixel data to a fixed colormap using equally spaced color values. Supports three dithering modes: none, ordered (Bayer 16×16), and Floyd-Steinberg error diffusion.

## Core Responsibilities
- Build an orthogonal colormap by dividing available colors across output components
- Precompute a `colorindex` lookup table mapping pixel values to nearest colormap entries (premultiplied by stride)
- Provide fast per-row pixel quantization (general N-component and 3-component fast paths)
- Implement ordered dithering using a static Bayer matrix
- Implement Floyd-Steinberg dithering with per-component error accumulation arrays
- Initialize and configure the `jpeg_color_quantizer` vtable registered on `cinfo->cquantize`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_cquantizer` | struct | Private quantizer state: colormap, colorindex, dither tables, FS error buffers, row state |
| `ODITHER_MATRIX` | typedef (int[16][16]) | 16×16 ordered dither value matrix |
| `ODITHER_MATRIX_PTR` | typedef (pointer) | Pointer to one row of an ODITHER_MATRIX |
| `FSERROR` | typedef (INT16/INT32) | Floyd-Steinberg accumulated error value, width depends on `BITS_IN_JSAMPLE` |
| `FSERRPTR` | typedef | FAR pointer to FSERROR array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `base_dither_matrix` | `const UINT8[16][16]` | static (file) | Bayer order-4 dither matrix; values 0–255, used to build scaled `ODITHER_MATRIX` per component |

## Key Functions / Methods

### select_ncolors
- **Signature:** `LOCAL int select_ncolors(j_decompress_ptr cinfo, int Ncolors[])`
- **Purpose:** Distributes `cinfo->desired_number_of_colors` across `out_color_components`, filling `Ncolors[]`.
- **Inputs:** `cinfo` (desired colors, component count, colorspace); `Ncolors[]` output array.
- **Outputs/Return:** Total color count (product of `Ncolors[]`).
- **Side effects:** Calls `ERREXIT1` on fatal conditions.
- **Calls:** `ERREXIT1`
- **Notes:** In RGB mode, increments G first, then R, then B to maximize perceptual quality. Requires ≥2 values per component.

### create_colormap
- **Signature:** `LOCAL void create_colormap(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and fills the equally-spaced colormap; stores it in `cquantize->sv_colormap`.
- **Inputs:** `cinfo` (component info, memory manager).
- **Outputs/Return:** void; sets `cquantize->sv_colormap` and `cquantize->sv_actual`.
- **Side effects:** Allocates JPOOL_IMAGE memory.
- **Calls:** `select_ncolors`, `output_value`, `alloc_sarray`, `TRACEMS4/TRACEMS1`

### create_colorindex
- **Signature:** `LOCAL void create_colorindex(j_decompress_ptr cinfo)`
- **Purpose:** Builds premultiplied lookup table `colorindex[component][pixelvalue]` → colormap index stride offset. Pads for ordered dithering if needed.
- **Inputs:** `cinfo` (dither mode, component count, memory manager).
- **Side effects:** Allocates JPOOL_IMAGE; sets `cquantize->is_padded`.
- **Calls:** `alloc_sarray`, `largest_input_value`

### make_odither_array
- **Signature:** `LOCAL ODITHER_MATRIX_PTR make_odither_array(j_decompress_ptr cinfo, int ncolors)`
- **Purpose:** Scales `base_dither_matrix` for a component with `ncolors` output levels; returns allocated matrix.
- **Outputs/Return:** Pointer to newly allocated `ODITHER_MATRIX`.
- **Side effects:** Allocates JPOOL_IMAGE.

### color_quantize / color_quantize3
- **Signature:** `METHODDEF void color_quantize(j_decompress_ptr, JSAMPARRAY, JSAMPARRAY, int)`
- **Purpose:** No-dither quantization. `color_quantize3` is an unrolled fast path for 3 components.
- **Side effects:** Writes colormap indices into `output_buf`.

### quantize_ord_dither / quantize3_ord_dither
- **Signature:** `METHODDEF void quantize_ord_dither(...)`
- **Purpose:** Ordered-dither quantization; advances `cquantize->row_index` per row.
- **Side effects:** Modifies `cquantize->row_index`.

### quantize_fs_dither
- **Signature:** `METHODDEF void quantize_fs_dither(...)`
- **Purpose:** Floyd-Steinberg dithering; propagates errors using standard 7/5/3/1 fractions, alternates scan direction per row.
- **Side effects:** Reads/writes `cquantize->fserrors[]`; toggles `cquantize->on_odd_row`.
- **Calls:** `jzero_far`, `GETJSAMPLE`, `RIGHT_SHIFT`

### start_pass_1_quant
- **Signature:** `METHODDEF void start_pass_1_quant(j_decompress_ptr cinfo, boolean is_pre_scan)`
- **Purpose:** Installs colormap on `cinfo`, selects the correct quantize function pointer based on `dither_mode`, and lazily creates dither tables or FS workspace if mode changed.
- **Side effects:** Writes `cinfo->colormap`, `cinfo->actual_number_of_colors`; may call `create_colorindex`, `create_odither_tables`, `alloc_fs_workspace`, `jzero_far`.

### jinit_1pass_quantizer
- **Signature:** `GLOBAL void jinit_1pass_quantizer(j_decompress_ptr cinfo)`
- **Purpose:** Module entry point — allocates `my_cquantizer`, wires vtable pointers, validates constraints, calls `create_colormap`/`create_colorindex`, optionally pre-allocates FS workspace.
- **Side effects:** Sets `cinfo->cquantize`; allocates JPOOL_IMAGE memory.
- **Calls:** `alloc_small`, `create_colormap`, `create_colorindex`, `alloc_fs_workspace`, `ERREXIT1`

## Control Flow Notes
- **Init:** `jinit_1pass_quantizer` called once during JPEG decompression setup; sets up all static tables.
- **Per-pass:** `start_pass_1_quant` called before each output pass; installs the chosen quantizer method pointer.
- **Per-row batch:** The installed `color_quantize` method pointer is called by the post-processing controller to convert rows of full-color samples to colormap indices.
- **Shutdown:** `finish_pass_1_quant` is a no-op; memory released via JPOOL_IMAGE pool destruction.

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG structs, `GETJSAMPLE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `ERREXIT*`, `TRACEMS*`, `jzero_far`
- `QUANT_1PASS_SUPPORTED` — compile-time guard; entire file is conditionally compiled
- `jzero_far` — defined elsewhere (jutils.c)
- `alloc_small`, `alloc_large`, `alloc_sarray` — provided by JPEG memory manager, defined elsewhere
