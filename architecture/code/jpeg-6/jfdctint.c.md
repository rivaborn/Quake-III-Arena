# code/jpeg-6/jfdctint.c

## File Purpose
Implements the slow-but-accurate integer forward Discrete Cosine Transform (FDCT) for the IJG JPEG library. It performs a separable 2-D 8×8 DCT using a scaled fixed-point version of the Loeffler-Ligtenberg-Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Perform the forward DCT on a single 8×8 block of `DCTELEM` samples in-place
- Execute two separable 1-D DCT passes: first across all 8 rows, then all 8 columns
- Apply scaled fixed-point integer arithmetic to avoid floating-point at runtime
- Scale outputs by `sqrt(8) * 2^PASS1_BITS` after pass 1; remove `PASS1_BITS` scaling after pass 2, leaving a net factor-of-8 scale (consumed by the quantization step in `jcdctmgr.c`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `DCTELEM` | typedef (int or INT32) | Element type of the 8×8 DCT working block; `int` for 8-bit samples, `INT32` for 12-bit |
| `INT32` | typedef | 32-bit signed integer used for intermediate accumulation |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_fdct_islow
- **Signature:** `GLOBAL void jpeg_fdct_islow(DCTELEM *data)`
- **Purpose:** Performs the complete in-place 2-D forward DCT on a single 64-element (8×8) block using slow/accurate scaled integer arithmetic.
- **Inputs:** `data` — pointer to a flat 64-element `DCTELEM` array (row-major, 8 rows × 8 cols); values expected in signed range ±`CENTERJSAMPLE`.
- **Outputs/Return:** `void`; `data[]` is overwritten with DCT coefficients scaled up by a factor of 8 relative to true DCT outputs.
- **Side effects:** Modifies `data` in-place; no heap allocation, no I/O, no global state touched.
- **Calls:** Macros only — `MULTIPLY`, `DESCALE`, `MULTIPLY16C16` (via `MULTIPLY`), `RIGHT_SHIFT` (via `DESCALE`); no function calls.
- **Notes:**
  - Pass 1 iterates rows (ctr from 7 down to 0), advancing `dataptr` by `DCTSIZE` each iteration.
  - Pass 2 iterates columns (ctr from 7 down to 0), advancing `dataptr` by 1 each iteration, accessing elements via `dataptr[DCTSIZE*k]` strides.
  - Pass 1 shift amount is `CONST_BITS - PASS1_BITS`; pass 2 uses `CONST_BITS + PASS1_BITS` to undo the pass-1 upscaling.
  - The factor-of-8 residual scale is intentionally left in the output for the quantization step (`jcdctmgr.c`) to absorb.
  - Guarded by `#ifdef DCT_ISLOW_SUPPORTED`; a compile-time assertion enforces `DCTSIZE == 8`.
  - For 8-bit samples (`BITS_IN_JSAMPLE == 8`): `CONST_BITS=13`, `PASS1_BITS=2`; for 12-bit: `PASS1_BITS=1` to prevent 32-bit overflow.

## Control Flow Notes
This file is called during JPEG **compression** only. `jcdctmgr.c` selects `jpeg_fdct_islow` when `dct_method == JDCT_ISLOW`, calling it once per 8×8 MCU block before quantization. It has no role in decompression, init, or frame/render loops.

## External Dependencies
- `jinclude.h` — system include abstraction, `MEMZERO`/`MEMCOPY`, `<stdio.h>`, `<string.h>`
- `jpeglib.h` — `DCTSIZE`, `INT32`, `BITS_IN_JSAMPLE`, `JSAMPLE`, JPEG object types
- `jdct.h` — `DCTELEM`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `CONST_SCALE`, `ONE`; declares `jpeg_fdct_islow` as `EXTERN`
- `SHIFT_TEMPS`, `RIGHT_SHIFT` — defined elsewhere (platform-specific, typically in `jmorecfg.h` or `jpegint.h`)
- `MULTIPLY16C16` — defined in `jdct.h`, platform-tunable for 16×16→32 multiply optimization
- `DCT_ISLOW_SUPPORTED` — compile-time feature flag, defined elsewhere in the build configuration
