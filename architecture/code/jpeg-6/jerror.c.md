# code/jpeg-6/jerror.c

## File Purpose
This is a Quake III Arena-adapted version of the IJG JPEG library's error-handling module. It replaces the standard Unix `stderr`-based error output with Quake's renderer interface (`ri.Error` and `ri.Printf`), integrating JPEG decode/encode errors into the engine's error and logging systems.

## Core Responsibilities
- Define and populate the JPEG standard message string table from `jerror.h`
- Implement the `error_exit` handler that calls `ri.Error(ERR_FATAL, ...)` on fatal JPEG errors
- Implement `output_message` to route JPEG messages through `ri.Printf`
- Implement `emit_message` with warning-level filtering and trace-level gating
- Implement `format_message` to produce formatted error strings from message codes and parameters
- Implement `reset_error_mgr` to clear error state between images
- Provide `jpeg_std_error` to wire all handler function pointers into a `jpeg_error_mgr`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `jpeg_error_mgr` | struct (defined in `jpeglib.h`) | Error manager object holding function pointers and error state |
| `j_common_ptr` | typedef (pointer) | Generic pointer to either compress or decompress instance |
| `J_MESSAGE_CODE` | enum (in `jerror.h`) | Enumerated JPEG error/trace message codes |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `jpeg_std_message_table` | `const char * const []` | global | Array of JPEG error/trace message strings indexed by `J_MESSAGE_CODE` |

## Key Functions / Methods

### error_exit
- **Signature:** `METHODDEF void error_exit(j_common_ptr cinfo)`
- **Purpose:** Fatal error handler; formats the error message, destroys the JPEG object, and aborts the engine.
- **Inputs:** `cinfo` — pointer to JPEG common struct with `err` populated
- **Outputs/Return:** None (does not return)
- **Side effects:** Calls `jpeg_destroy(cinfo)` to free JPEG allocations; calls `ri.Error(ERR_FATAL, ...)` which terminates the engine
- **Calls:** `cinfo->err->format_message`, `jpeg_destroy`, `ri.Error`
- **Notes:** Replaces the default IJG behavior of calling `exit()`; uses `longjmp`-capable override point if needed by callers

### output_message
- **Signature:** `METHODDEF void output_message(j_common_ptr cinfo)`
- **Purpose:** Routes a formatted JPEG message to the renderer's print channel.
- **Inputs:** `cinfo`
- **Outputs/Return:** None
- **Side effects:** Calls `ri.Printf(PRINT_ALL, ...)`
- **Calls:** `cinfo->err->format_message`, `ri.Printf`

### emit_message
- **Signature:** `METHODDEF void emit_message(j_common_ptr cinfo, int msg_level)`
- **Purpose:** Decides whether a warning or trace message is printed based on `msg_level` and `trace_level`.
- **Inputs:** `cinfo`, `msg_level` (-1 = warning, 0+ = trace levels)
- **Outputs/Return:** None
- **Side effects:** Increments `err->num_warnings` for warnings; calls `output_message` when policy allows
- **Notes:** Only the first warning is shown unless `trace_level >= 3`

### format_message
- **Signature:** `METHODDEF void format_message(j_common_ptr cinfo, char *buffer)`
- **Purpose:** Formats a JPEG error message string (from the message table + parameters) into `buffer`.
- **Inputs:** `cinfo`, `buffer` (at least `JMSG_LENGTH_MAX` bytes)
- **Outputs/Return:** Fills `buffer` in-place
- **Side effects:** None
- **Notes:** Detects `%s` vs integer parameters; uses `sprintf` directly — no bounds checking beyond caller-supplied buffer size

### reset_error_mgr
- **Signature:** `METHODDEF void reset_error_mgr(j_common_ptr cinfo)`
- **Purpose:** Resets `num_warnings` and `msg_code` to zero at the start of a new image; preserves `trace_level`.
- **Side effects:** Modifies `cinfo->err` fields

### jpeg_std_error
- **Signature:** `GLOBAL struct jpeg_error_mgr *jpeg_std_error(struct jpeg_error_mgr *err)`
- **Purpose:** Initializes all function pointers and state in a `jpeg_error_mgr` to their standard defaults.
- **Inputs:** Caller-allocated `err`
- **Outputs/Return:** Returns `err` after initialization
- **Side effects:** Writes all fields of `*err`; sets `jpeg_message_table` to `jpeg_std_message_table`
- **Notes:** Entry point called by the application before creating a JPEG compress/decompress object

## Control Flow Notes
This file is not part of the frame/render loop directly. It is called during image loading (JPEG decompression) triggered by `R_FindImageFile`/`R_LoadJPG` in the renderer. `jpeg_std_error` is called at setup time. `error_exit` can fire during any JPEG decode operation, immediately halting the engine via `ri.Error(ERR_FATAL)`.

## External Dependencies
- `jinclude.h` — platform-specific includes and memory macros
- `jpeglib.h` — JPEG library types and struct definitions
- `jversion.h` — version string constants embedded in the message table
- `jerror.h` — message code enum and `JMESSAGE` macro (included twice via X-macro pattern)
- `../renderer/tr_local.h` — provides `ri` (`refimport_t`) for `ri.Error` and `ri.Printf`; **defined elsewhere** in the renderer module
- `jpeg_destroy` — defined elsewhere in the IJG library (`jcomapi.c`)
- `ri.Error`, `ri.Printf` — defined elsewhere; renderer import table populated at renderer initialization
