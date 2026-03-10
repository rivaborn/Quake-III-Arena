# code/jpeg-6/jquant2.c

## File Purpose
Implements 2-pass color quantization (color mapping) for the IJG JPEG decompressor. Pass 1 builds a color usage histogram; pass 2 maps each pixel to the nearest entry in a custom colormap derived via median-cut, with optional Floyd-Steinberg dithering.

## Core Responsibilities
- Accumulate a 3D RGB histogram during prescan (pass 1)
- Run Heckbert/median-cut box-splitting to select a representative colormap
- Build a lazy-filled inverse colormap (histogram reused as lookup cache)
- Map pixels to colormap entries without dithering (`pass2_no_dither`)
- Map pixels to colormap entries with Floyd-Steinberg dithering (`pass2_fs_dither`)
- Initialize and own the error-limiting table for F-S dithering
- Register itself as the `cquantize` subobject on `j_decompress_ptr`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_cquantizer` | struct | Private quantizer state: histogram, colormap storage, F-S error buffer, error limiter, odd-row flag |
| `box` | struct | Median-cut box: C0/C1/C2 min/max bounds, scaled 2-norm volume, nonzero-cell population count |
| `histcell` | typedef (UINT16) | Single histogram/inverse-cmap cell |
| `hist1d` / `hist2d` / `hist3d` | typedef arrays/pointers | Three-level indirected 3-D histogram structure (DOS near/far memory model) |
| `FSERROR` / `LOCFSERROR` | typedef (INT16/int) | Floyd-Steinberg error accumulator types |

## Global / File-Static State
None.

## Key Functions / Methods

### prescan_quantize
- **Signature:** `METHODDEF void prescan_quantize(j_decompress_ptr cinfo, JSAMPARRAY input_buf, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Pass 1 — increment the 3-D histogram for every input pixel.
- **Inputs:** `input_buf` rows of RGB pixels; `num_rows` count.
- **Outputs/Return:** None (histogram mutated in place; `output_buf` unused).
- **Side effects:** Writes `cquantize->histogram`; saturates at UINT16 max.
- **Calls:** None beyond array indexing macros.
- **Notes:** Overflow check is "undo decrement if wrapped to ≤ 0" pattern.

### select_colors
- **Signature:** `LOCAL void select_colors(j_decompress_ptr cinfo, int desired_colors)`
- **Purpose:** Orchestrates median-cut: allocates box list, calls `update_box`, `median_cut`, then `compute_color` per box.
- **Inputs:** `desired_colors` — target colormap size.
- **Outputs/Return:** Fills `cinfo->colormap`; sets `cinfo->actual_number_of_colors`.
- **Side effects:** Allocates small JPOOL_IMAGE block; emits trace message.
- **Calls:** `update_box`, `median_cut`, `compute_color`, `alloc_small`, `TRACEMS1`.

### median_cut
- **Signature:** `LOCAL int median_cut(j_decompress_ptr cinfo, boxptr boxlist, int numboxes, int desired_colors)`
- **Purpose:** Iteratively splits boxes by longest scaled axis until `desired_colors` boxes exist.
- **Inputs:** `boxlist`, current `numboxes`, target `desired_colors`.
- **Outputs/Return:** Final box count.
- **Side effects:** Calls `update_box` on each new pair; modifies boxlist in place.
- **Calls:** `find_biggest_color_pop`, `find_biggest_volume`, `update_box`.
- **Notes:** Uses population criterion for first half, volume criterion thereafter.

### fill_inverse_cmap
- **Signature:** `LOCAL void fill_inverse_cmap(j_decompress_ptr cinfo, int c0, int c1, int c2)`
- **Purpose:** Fills one subbox of the histogram cache with best-match colormap indices.
- **Inputs:** Histogram cell coordinates `c0/c1/c2` identifying the update box.
- **Outputs/Return:** Writes `histogram[...] = colormap_index + 1` for all cells in the box.
- **Side effects:** Mutates histogram (now used as inverse-cmap cache).
- **Calls:** `find_nearby_colors`, `find_best_colors`.

### pass2_no_dither
- **Signature:** `METHODDEF void pass2_no_dither(j_decompress_ptr cinfo, JSAMPARRAY input_buf, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Pass 2 pixel mapping without dithering; cache-miss triggers `fill_inverse_cmap`.
- **Side effects:** Reads/writes histogram cache; fills output rows with 8-bit colormap indices.
- **Calls:** `fill_inverse_cmap`.

### pass2_fs_dither
- **Signature:** `METHODDEF void pass2_fs_dither(j_decompress_ptr cinfo, JSAMPARRAY input_buf, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Pass 2 with Floyd-Steinberg dithering; alternates scan direction per row.
- **Side effects:** Reads/writes `cquantize->fserrors`, `on_odd_row`; fills output with colormap indices.
- **Calls:** `fill_inverse_cmap`.
- **Notes:** Uses `error_limiter` table and `sample_range_limit` to clamp corrected values.

### jinit_2pass_quantizer
- **Signature:** `GLOBAL void jinit_2pass_quantizer(j_decompress_ptr cinfo)`
- **Purpose:** Module init: allocates `my_cquantizer`, histogram, colormap storage, and optionally F-S buffers; installs method pointers.
- **Side effects:** Allocates JPOOL_IMAGE memory; sets `cinfo->cquantize`; calls `init_error_limit`.
- **Calls:** `alloc_small`, `alloc_large`, `alloc_sarray`, `init_error_limit`, `ERREXIT`.
- **Notes:** Errors if `out_color_components != 3`.

### Notes on minor helpers
- `find_biggest_color_pop` / `find_biggest_volume` — linear scans of boxlist for split candidate selection.
- `update_box` — tightens box bounds and recomputes 2-norm volume and population.
- `compute_color` — weighted-mean colormap entry for one box.
- `find_nearby_colors` — Heckbert locality filter to prune colormap candidates per subbox.
- `find_best_colors` — Thomas incremental distance scan to assign best color to every cell in a subbox.
- `init_error_limit` — builds piecewise-linear clamping LUT for F-S error values.
- `finish_pass1` / `finish_pass2` / `start_pass_2_quant` / `new_color_map_2_quant` — pass lifecycle callbacks.

## Control Flow Notes
`jinit_2pass_quantizer` is called at decompressor init. `start_pass_2_quant` is called before each pass: with `is_pre_scan=TRUE` it wires `prescan_quantize`; with `FALSE` it wires `pass2_fs_dither` or `pass2_no_dither`. `finish_pass1` triggers colormap selection between the two passes. The file is entirely decompression-side; no encoder involvement.

## External Dependencies
- `jinclude.h` — system includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG object definitions, `METHODDEF`, `LOCAL`, `GLOBAL`, `ERREXIT`, `TRACEMS1`, `jzero_far`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `GETJSAMPLE`, `MAXJSAMPLE`, `BITS_IN_JSAMPLE`
- `RGB_RED`, `RGB_GREEN`, `RGB_BLUE` — defined in `jmorecfg.h`; control component ordering macros
- `cinfo->mem->alloc_small/alloc_large/alloc_sarray` — defined in memory manager, called via function pointers
