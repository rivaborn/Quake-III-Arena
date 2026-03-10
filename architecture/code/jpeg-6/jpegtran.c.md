# code/jpeg-6/jpegtran.c

## File Purpose
A standalone command-line application for lossless JPEG transcoding. It reads a JPEG file as raw DCT coefficients and rewrites it with different encoding parameters (progressive, arithmetic coding, restart intervals, etc.) without a full decode/re-encode cycle.

## Core Responsibilities
- Parse command-line switches to configure a JPEG compression context
- Open input/output files (or fall back to stdin/stdout)
- Decompress source JPEG into DCT coefficient arrays (lossless read)
- Copy critical parameters from source to destination compressor
- Re-compress using DCT arrays directly, preserving image quality
- Clean up all JPEG objects and file handles on exit

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `jpeg_decompress_struct` (`srcinfo`) | struct | IJG decompressor context for reading input |
| `jpeg_compress_struct` (`dstinfo`) | struct | IJG compressor context for writing output |
| `jpeg_error_mgr` (`jsrcerr`, `jdsterr`) | struct | Error manager instances for each codec object |
| `jvirt_barray_ptr *` (`coef_arrays`) | typedef/pointer array | Array of virtual block arrays holding raw DCT coefficients |
| `cdjpeg_progress_mgr` (`progress`) | struct | Optional progress-reporting hook (conditional on `PROGRESS_REPORT`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `progname` | `const char *` | static (file) | Program name used in error/usage messages |
| `outfilename` | `char *` | static (file) | Output filename set by `-outfile` switch; reset per input file |

## Key Functions / Methods

### usage
- **Signature:** `LOCAL void usage(void)`
- **Purpose:** Prints usage information to stderr and exits with `EXIT_FAILURE`.
- **Inputs:** None (reads global `progname`; compile-time feature macros control which switches are listed).
- **Outputs/Return:** No return; terminates process.
- **Side effects:** Writes to `stderr`; calls `exit()`.
- **Calls:** `fprintf`, `exit`
- **Notes:** Feature-conditional blocks (`#ifdef`) gate display of `-optimize`, `-progressive`, `-arithmetic`, `-scans`.

---

### parse_switches
- **Signature:** `LOCAL int parse_switches(j_compress_ptr cinfo, int argc, char **argv, int last_file_arg_seen, boolean for_real)`
- **Purpose:** Iterates `argv` to configure `cinfo` (the destination compressor). Designed for two-pass use: a dummy pass (`for_real=FALSE`) to locate file arguments, then a real pass after the source is opened.
- **Inputs:** `cinfo` — destination compress struct; `argc`/`argv` — raw command line; `last_file_arg_seen` — skip already-processed file args; `for_real` — whether expensive operations (scan-script loading) should execute.
- **Outputs/Return:** `int` — `argv` index of the first non-switch argument (file name).
- **Side effects:** Mutates `cinfo` fields (`arith_code`, `optimize_coding`, `restart_interval`, `restart_in_rows`, `mem->max_memory_to_use`, `err->trace_level`); sets global `outfilename`; may call `jpeg_simple_progression` or `read_scan_script`; calls `exit` on bad input.
- **Calls:** `keymatch`, `sscanf`, `jpeg_simple_progression`, `read_scan_script`, `fprintf`, `exit`, `usage`
- **Notes:** `-scans` file reading is deferred until `for_real` to allow `-progressive` to be processed first. `printed_version` is a function-static boolean guarding one-time version output.

---

### main
- **Signature:** `GLOBAL int main(int argc, char **argv)`
- **Purpose:** Application entry point; orchestrates the full transcode pipeline.
- **Inputs:** Standard `argc`/`argv`.
- **Outputs/Return:** Exits with `EXIT_SUCCESS`, `EXIT_WARNING` (if warnings occurred), or `EXIT_FAILURE`.
- **Side effects:** Creates and destroys IJG codec objects; opens/closes files; reads and writes to filesystem or stdio streams; may install a signal catcher.
- **Calls:** `jpeg_std_error`, `jpeg_create_decompress`, `jpeg_create_compress`, `enable_signal_catcher`, `parse_switches` (×2), `fopen`, `read_stdin`, `write_stdout`, `start_progress_monitor`, `jpeg_stdio_src`, `jpeg_read_header`, `jpeg_read_coefficients`, `jpeg_copy_critical_parameters`, `jpeg_stdio_dest`, `jpeg_write_coefficients`, `jpeg_finish_compress`, `jpeg_destroy_compress`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, `fclose`, `end_progress_monitor`, `exit`
- **Notes:** `parse_switches` is called twice — first to locate file arguments (dummy pass), then again after codec initialization for the real parameter application. Source comments are acknowledged as missing (`/* ought to copy source comments here... */`). Exit code is non-zero if either error manager recorded warnings.

## Control Flow Notes

This is a batch-processing tool, not a frame-driven engine component. Execution is linear: init → parse (dummy) → open files → decode to DCT coefficients → parse (real) → encode from coefficients → teardown → exit. It does not participate in Quake III's frame loop; it is a build-tool utility shipped in the `jpeg-6/` subdirectory.

## External Dependencies

- **`cdjpeg.h`** — IJG common application declarations; provides `keymatch`, `read_stdin`, `write_stdout`, `read_scan_script`, `start_progress_monitor`, `end_progress_monitor`, `enable_signal_catcher`, `READ_BINARY`, `WRITE_BINARY`, `TWO_FILE_COMMANDLINE`
- **`jversion.h`** — `JVERSION`, `JCOPYRIGHT` string macros
- **Defined elsewhere (IJG library):** `jpeg_create_decompress`, `jpeg_create_compress`, `jpeg_std_error`, `jpeg_read_header`, `jpeg_read_coefficients`, `jpeg_copy_critical_parameters`, `jpeg_write_coefficients`, `jpeg_finish_compress`, `jpeg_destroy_compress`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, `jpeg_stdio_src`, `jpeg_stdio_dest`, `jpeg_simple_progression`, `j_compress_ptr`, `jvirt_barray_ptr`
