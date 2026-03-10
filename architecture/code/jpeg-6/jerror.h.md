# code/jpeg-6/jerror.h

## File Purpose
Defines all error and trace message codes for the IJG JPEG library as a `J_MESSAGE_CODE` enum, and provides a set of convenience macros for emitting fatal errors, warnings, and trace/debug messages through the JPEG library's error manager vtable.

## Core Responsibilities
- Declares the `J_MESSAGE_CODE` enum by expanding `JMESSAGE` macros into enum values
- Provides `ERREXIT`/`ERREXIT1–4`/`ERREXITS` macros for fatal error dispatch (calls `error_exit` function pointer)
- Provides `WARNMS`/`WARNMS1–2` macros for non-fatal/corrupt-data warnings (calls `emit_message` at level -1)
- Provides `TRACEMS`/`TRACEMS1–8`/`TRACEMSS` macros for informational and debug tracing (calls `emit_message` at caller-supplied level)
- Supports dual-inclusion pattern: first include builds the enum, second include (with `JMESSAGE` defined externally) builds a string table

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `J_MESSAGE_CODE` | enum (typedef) | Enumeration of all library message codes; values range from `JMSG_NOMESSAGE` through `JMSG_LASTMSGCODE` (sentinel) |

## Global / File-Static State
None.

## Key Functions / Methods
No functions are defined in this header. All callable logic is expressed as macros.

### ERREXIT / ERREXIT1–4 / ERREXITS
- Signature: Macro(`cinfo`, `code` [, p1..p4 or str])
- Purpose: Trigger a fatal JPEG error; sets `msg_code` and optional integer/string parameters on the error manager, then invokes the `error_exit` function pointer — which typically `longjmp`s or `exit()`s.
- Inputs: `cinfo` — pointer to `j_compress_ptr` or `j_decompress_ptr`; `code` — `J_MESSAGE_CODE`; optional integer or string parameters.
- Outputs/Return: No return (does not continue; `error_exit` is `noreturn` by convention).
- Side effects: Writes to `cinfo->err->msg_code`, `cinfo->err->msg_parm`; calls `error_exit` via vtable.
- Calls: `(*(cinfo)->err->error_exit)((j_common_ptr)(cinfo))`
- Notes: `ERREXITS` uses `strncpy` into `msg_parm.s` for a string parameter variant.

### WARNMS / WARNMS1 / WARNMS2
- Signature: Macro(`cinfo`, `code` [, p1, p2])
- Purpose: Emit a non-fatal warning for corrupt but recoverable data.
- Inputs: `cinfo`, `code`, up to 2 integer parameters.
- Outputs/Return: Void expression.
- Side effects: Writes `msg_code`/`msg_parm`; calls `emit_message` with level `-1`.
- Calls: `(*(cinfo)->err->emit_message)((j_common_ptr)(cinfo), -1)`
- Notes: Level `-1` is the IJG convention for warnings.

### TRACEMS / TRACEMS1–8 / TRACEMSS
- Signature: Macro(`cinfo`, `lvl`, `code` [, p1..p8 or str])
- Purpose: Emit informational or debug trace messages at a caller-specified verbosity level.
- Inputs: `cinfo`, integer trace level `lvl`, `code`, up to 8 integer parameters or one string.
- Outputs/Return: Void expression or `do{…}while(0)` statement.
- Side effects: Writes `msg_code`/`msg_parm`; calls `emit_message` at `lvl`.
- Calls: `(*(cinfo)->err->emit_message)((j_common_ptr)(cinfo), (lvl))`
- Notes: `TRACEMS3` and above use the `MAKESTMT` wrapper to safely sequence multiple assignments before the call.

## Control Flow Notes
This is a pure header; it participates in library init only in the sense that `jpeg_std_error()` (defined in `jerror.c`) populates `jpeg_error_mgr` with function pointers that these macros invoke. At runtime, every error/warn/trace site in the library calls through the vtable, so the error manager is always exercised during codec operation.

## External Dependencies
- No `#include` directives in this file.
- `JCOPYRIGHT`, `JVERSION`: string macros, defined in `jversion.h` (included elsewhere).
- `JMSG_STR_PARM_MAX`: integer constant, defined in `jpeglib.h`.
- `j_common_ptr`, `j_compress_ptr`, `j_decompress_ptr`: typedefs defined in `jpeglib.h`.
- `strncpy`: standard C library, used in `ERREXITS` and `TRACEMSS`.
- `error_exit`, `emit_message`: function pointer fields on `jpeg_error_mgr`, defined/populated in `jerror.c`.
