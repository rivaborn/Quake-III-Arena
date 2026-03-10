# code/jpeg-6/jfdctfst.c

## File Purpose
Implements the fast, lower-accuracy integer forward Discrete Cosine Transform (DCT) for the IJG JPEG library. It applies the Arai, Agui & Nakajima (AA&N) scaled DCT algorithm to an 8×8 block of DCT elements in-place, using only 5 multiplies and 29 adds per 1-D pass.

## Core Responsibilities
- Perform a 2-pass (row then column) separable 8-point 1-D forward DCT on a single 8×8 block
- Encode fixed-point multiplicative constants at 8 fractional bits (`CONST_BITS = 8`)
- Provide an optionally less-accurate descale path (`USE_ACCURATE_ROUNDING` not defined → plain right-shift)
- Guard the entire implementation behind `#ifdef DCT_IFAST_SUPPORTED`
- Write scaled DCT coefficients back into the input buffer in-place (output×8 convention per JPEG spec)

## Key Types / Data Structures
None — operates on the externally-defined `DCTELEM` array type (aliased to `int` for 8-bit samples via `jdct.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_fdct_ifast
- **Signature:** `GLOBAL void jpeg_fdct_ifast(DCTELEM * data)`
- **Purpose:** Performs the complete forward DCT on one 8×8 block using the AA&N fast integer algorithm.
- **Inputs:** `data` — pointer to a 64-element `DCTELEM` array in row-major order, values expected in the signed range ±CENTERJSAMPLE.
- **Outputs/Return:** `void`; results written back in-place to `data`, scaled up by 8 (standard IJG convention).
- **Side effects:** Modifies `data` in-place; no heap allocation, no I/O, no global state.
- **Calls:** `MULTIPLY` macro (which expands to `DESCALE` / `RIGHT_SHIFT`); no other functions.
- **Notes:**
  - Pass 1 iterates over 8 rows (`dataptr += DCTSIZE` each iteration).
  - Pass 2 iterates over 8 columns (`dataptr++` each iteration), reusing the identical butterfly/rotation logic.
  - `MULTIPLY(var, const)` descales immediately after each fixed-point multiply, trading accuracy for reduced intermediate precision requirements (16-bit arithmetic suffices everywhere except inside the multiply itself).
  - Without `USE_ACCURATE_ROUNDING`, `DESCALE` is replaced by a plain `RIGHT_SHIFT` (no +0.5 rounding bias), introducing half-ULP errors in roughly half of all descale operations.
  - The AA&N rotator is modified from the reference figure (P&M fig. 4-8) to eliminate extra negations, reusing `z5` as a shared term for both `z2` and `z4`.
  - Hard requirement: `DCTSIZE == 8`; a deliberate compile-time syntax error fires otherwise.

## Control Flow Notes
This file is a pure computation module with no init/shutdown lifecycle. It is called during the compression pipeline by `jcdctmgr.c`, which selects among `jpeg_fdct_islow`, `jpeg_fdct_ifast`, and `jpeg_fdct_float` based on `cinfo->dct_method`. It runs once per 8×8 MCU block before quantization.

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — top-level JPEG library types and constants (`DCTSIZE`, `DCTSIZE2`)
- `jdct.h` — `DCTELEM` typedef, `DESCALE`, `RIGHT_SHIFT`, `FIX`, `SHIFT_TEMPS`, forward DCT extern declarations
- `DCTELEM`, `INT32`, `RIGHT_SHIFT`, `SHIFT_TEMPS` — defined elsewhere (`jmorecfg.h`, `jdct.h`, compiler/platform headers)
- `DCT_IFAST_SUPPORTED` — configuration macro defined in `jconfig.h`
