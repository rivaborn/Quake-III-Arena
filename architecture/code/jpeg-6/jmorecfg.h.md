# code/jpeg-6/jmorecfg.h

## File Purpose
Platform-portability and capability configuration header for the Independent JPEG Group (IJG) JPEG library. It defines primitive typedefs, compile-time capability switches, and machine-dependent tuning macros used throughout the JPEG codec.

## Core Responsibilities
- Define `JSAMPLE` (pixel sample type) and `JCOEF` (DCT coefficient type) based on bit-depth setting
- Provide portable integer typedefs (`UINT8`, `UINT16`, `INT16`, `INT32`, `JDIMENSION`, `JOCTET`)
- Guard against `unsigned char` / `char` signedness portability issues via `GETJSAMPLE`/`GETJOCTET` macros
- Declare function-linkage macros (`METHODDEF`, `LOCAL`, `GLOBAL`, `EXTERN`)
- Enable/disable encoder and decoder feature modules at compile time
- Configure RGB scanline channel ordering and pixel stride
- Provide performance hints: `INLINE`, `MULTIPLIER`, `FAST_FLOAT`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `JSAMPLE` | typedef | Single pixel channel value (8-bit: `unsigned char`/`char`; 12-bit: `short`) |
| `JCOEF` | typedef | DCT frequency coefficient (`short`) |
| `JOCTET` | typedef | Compressed bitstream byte (`unsigned char`/`char`) |
| `UINT8` | typedef | Unsigned 8-bit integer for tables |
| `UINT16` | typedef | Unsigned 16-bit integer for tables |
| `INT16` | typedef | Signed 16-bit integer |
| `INT32` | typedef | Signed 32-bit integer (`long`) |
| `JDIMENSION` | typedef | Image width/height (`unsigned int`, max 64K) |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure configuration/typedef header with no function definitions.

## Control Flow Notes
Purely declarative. Included early in every JPEG source file (via `jpeglib.h` or `jinclude.h`). No runtime participation in init/frame/render/shutdown cycles. Capability `#define`/`#undef` directives under `JPEG_INTERNAL_OPTIONS` gate which `.c` source files are compiled and linked — effectively a static feature-flag system.

**Notable Quake III deviation from stock IJG defaults:**
- `RGB_PIXELSIZE` is set to **4** (not 3), meaning scanline elements carry a dummy/alpha byte — aligns with Quake's 32-bit RGBA texture format. This disables color quantization compatibility per the comment.
- `DCT_ISLOW_SUPPORTED` and `DCT_IFAST_SUPPORTED` are `#undef`'d; only `DCT_FLOAT_SUPPORTED` is active, preferring floating-point DCT (faster on target HW).
- All decoder advanced features (progressive JPEG, multi-scan, block smoothing, IDCT scaling, upsampling merging, color quantization) are `#undef`'d — the library is stripped to a minimal decode path.

## External Dependencies
- No includes of its own.
- Consumed by: `jpeglib.h`, and transitively all `j*.c` translation units in `code/jpeg-6/`.
- Conditioned on external macros: `HAVE_UNSIGNED_CHAR`, `HAVE_UNSIGNED_SHORT`, `CHAR_IS_UNSIGNED`, `NEED_FAR_POINTERS`, `XMD_H`, `HAVE_PROTOTYPES`, `HAVE_BOOLEAN`, `JPEG_INTERNALS`, `__GNUC__` — all expected to be set (or absent) by `jconfig.h` or the build system.
