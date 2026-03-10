# code/jpeg-6/jccoefct.c

## File Purpose
Implements the coefficient buffer controller for JPEG compression. It sits between the forward-DCT stage and entropy encoding, managing how DCT coefficient blocks are collected, buffered, and fed to the entropy encoder. It is the top-level controller of the JPEG compressor proper.

## Core Responsibilities
- Initialize and manage the coefficient buffer (single-MCU or full-image virtual arrays)
- Dispatch the correct `compress_data` function pointer based on pass mode
- Run forward DCT on input sample rows and accumulate coefficient blocks into MCUs
- Handle padding (dummy blocks) at right and bottom image edges
- Support single-pass (pass-through) and multi-pass (Huffman optimization / multi-scan) compression
- Suspend and resume mid-row if the entropy encoder stalls

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_coef_controller` | struct | Private extension of `jpeg_c_coef_controller`; holds iMCU row counters, MCU buffer pointers, and optional virtual block arrays |
| `my_coef_ptr` | typedef (pointer) | Convenience alias for `my_coef_controller *` |

## Global / File-Static State
None.

## Key Functions / Methods

### start_iMCU_row
- **Signature:** `LOCAL void start_iMCU_row(j_compress_ptr cinfo)`
- **Purpose:** Resets within-iMCU-row counters at the start of each new iMCU row. Computes `MCU_rows_per_iMCU_row` (1 for interleaved scans; `v_samp_factor` or `last_row_height` for non-interleaved).
- **Inputs:** `cinfo` — active compression context
- **Outputs/Return:** void
- **Side effects:** Writes `coef->MCU_rows_per_iMCU_row`, `coef->mcu_ctr`, `coef->MCU_vert_offset`
- **Calls:** None
- **Notes:** Called at init and after each completed iMCU row.

### start_pass_coef
- **Signature:** `METHODDEF void start_pass_coef(j_compress_ptr cinfo, J_BUF_MODE pass_mode)`
- **Purpose:** Initializes state for a compression pass and sets `coef->pub.compress_data` to the appropriate function (`compress_data`, `compress_first_pass`, or `compress_output`) based on `pass_mode`.
- **Inputs:** `cinfo`, `pass_mode` (JBUF_PASS_THRU / JBUF_SAVE_AND_PASS / JBUF_CRANK_DEST)
- **Outputs/Return:** void
- **Side effects:** Resets `iMCU_row_num`; sets function pointer; errors on invalid mode/buffer mismatch.
- **Calls:** `start_iMCU_row`, `ERREXIT`

### compress_data
- **Signature:** `METHODDEF boolean compress_data(j_compress_ptr cinfo, JSAMPIMAGE input_buf)`
- **Purpose:** Single-pass compression. For one iMCU row, runs forward DCT per component block row, pads right/bottom edges with dummy blocks, then calls the entropy encoder per MCU. Returns FALSE on encoder suspension.
- **Inputs:** `cinfo`, `input_buf` (per-component sample planes)
- **Outputs/Return:** TRUE = iMCU row complete; FALSE = suspended
- **Side effects:** Writes DCT coefficients into `coef->MCU_buffer`; advances `iMCU_row_num`; updates suspension counters on FALSE return.
- **Calls:** `cinfo->fdct->forward_DCT`, `cinfo->entropy->encode_mcu`, `jzero_far`, `start_iMCU_row`
- **Notes:** On suspension the MCU is re-DCT'd on retry (noted as a minor inefficiency in comments).

### compress_first_pass *(FULL_COEF_BUFFER_SUPPORTED)*
- **Signature:** `METHODDEF boolean compress_first_pass(j_compress_ptr cinfo, JSAMPIMAGE input_buf)`
- **Purpose:** First pass of multi-pass compression. DCTs all components and stores coefficients into virtual block arrays, filling right/bottom dummy blocks. Then delegates to `compress_output` to emit entropy data.
- **Inputs:** `cinfo`, `input_buf`
- **Outputs/Return:** Return value from `compress_output`
- **Side effects:** Writes to virtual arrays via `cinfo->mem->access_virt_barray`; dummy rows/columns constructed in virtual storage.
- **Calls:** `cinfo->mem->access_virt_barray`, `cinfo->fdct->forward_DCT`, `jzero_far`, `compress_output`
- **Notes:** On suspension, all DCT work is repeated next call (no partial-save mechanism).

### compress_output *(FULL_COEF_BUFFER_SUPPORTED)*
- **Signature:** `METHODDEF boolean compress_output(j_compress_ptr cinfo, JSAMPIMAGE input_buf)`
- **Purpose:** Subsequent passes (or second half of first pass): reads MCU blocks from virtual arrays and feeds them to the entropy encoder.
- **Inputs:** `cinfo`, `input_buf` (ignored)
- **Outputs/Return:** TRUE = iMCU row complete; FALSE = suspended
- **Side effects:** Advances `iMCU_row_num` on completion; updates suspension counters on FALSE.
- **Calls:** `cinfo->mem->access_virt_barray`, `cinfo->entropy->encode_mcu`, `start_iMCU_row`

### jinit_c_coef_controller
- **Signature:** `GLOBAL void jinit_c_coef_controller(j_compress_ptr cinfo, boolean need_full_buffer)`
- **Purpose:** Allocates and wires up the coefficient controller. If `need_full_buffer`, allocates per-component virtual block arrays padded to sampling-factor multiples. Otherwise allocates a single contiguous MCU-sized block buffer.
- **Inputs:** `cinfo`, `need_full_buffer`
- **Outputs/Return:** void
- **Side effects:** Allocates memory via `cinfo->mem`; sets `cinfo->coef`; registers `start_pass_coef`.
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->alloc_large`, `cinfo->mem->request_virt_barray`, `jround_up`, `ERREXIT`

## Control Flow Notes
`jinit_c_coef_controller` is called during compression initialization. Each compression pass begins with `start_pass_coef` (called by the master controller), which selects the data function. The compressor master then calls `compress_data` (or `compress_first_pass` / `compress_output`) once per iMCU row until all rows are consumed.

## External Dependencies
- `jinclude.h` — system includes, SIZEOF, MEMZERO macros
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jvirt_barray_ptr`, `JBLOCKROW`, `J_BUF_MODE`, `JDIMENSION`, etc.
- **Defined elsewhere:** `jzero_far`, `jround_up` (utility routines); `cinfo->fdct->forward_DCT` (forward DCT module); `cinfo->entropy->encode_mcu` (entropy encoder); `cinfo->mem->*` (memory manager); `ERREXIT` (error handler macro)
