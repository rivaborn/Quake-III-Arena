# Subsystem Overview

## Purpose
`code/jpeg-6` is a vendored, Quake III Arena-adapted build of the Independent JPEG Group (IJG) libjpeg version 6, dated August 2, 1995. It provides full JPEG compression and decompression functionality, integrated into the engine by replacing the standard error handler and memory allocator with renderer-native equivalents (`ri.Error`, `ri.Printf`, `ri.Malloc`, `ri.Free`). The subsystem is used primarily to load JPEG-encoded texture images from the Quake virtual filesystem into heap-allocated pixel buffers.

---

## Key Files

| File | Role |
|---|---|
| `jload.c` | Engine integration entry point; bridges `FS_FOpenFileRead`/`Z_Malloc` with the libjpeg decompression API |
| `jerror.c` | Quake-adapted error handler; routes fatal errors to `ri.Error` and messages to `ri.Printf` |
| `jmemnobs.c` | Quake-adapted memory backend; routes all JPEG allocations through `ri.Malloc`/`ri.Free` |
| `jdatasrc.c` | Memory-buffer JPEG source manager; feeds compressed bytes from an in-memory pointer |
| `jdatadst.c` | stdio-based JPEG destination manager; routes compressed output to a `FILE*` stream |
| `jpeglib.h` | Public API header; defines all compress/decompress context structs and entry points |
| `jpegint.h` | Internal vtable structs and `jinit_*` prototypes wiring the pipeline |
| `jmorecfg.h` | Primitive typedefs, feature switches, and performance tuning macros |
| `jconfig.h` | Platform configuration header (Watcom/DOS target); feature-flag source for all other headers |
| `jinclude.h` | Portability shim; selects system headers and defines `MEMZERO`/`MEMCOPY`/`SIZEOF`/`JFREAD`/`JFWRITE` |
| `jerror.h` | Error/trace message code enum and `ERREXIT`/`WARNMS`/`TRACEMS` macro family |
| `jdct.h` | Private DCT/IDCT subsystem header; typedefs, multiplier types, fixed-point arithmetic macros, all DCT extern declarations |
| `jchuff.h` | Shared Huffman encoding structs and prototypes for sequential and progressive encoders |
| `jdhuff.h` | Shared Huffman decoding structs, bit-reader state, and performance-critical decode macros |
| `jmemsys.h` | System-dependent memory backend interface contract |
| `jversion.h` | Library version and copyright string macros |
| `jutils.c` | Shared zigzag/natural-order tables, integer math helpers, array copy/zero utilities |
| `jcomapi.c` | Shared compress/decompress lifecycle (abort, destroy) and table allocators |
| `jcapimin.c` / `jcapistd.c` | Minimum and standard compression public API entry points |
| `jcinit.c` | Compression pipeline wiring; calls all `jinit_*` routines in correct order |
| `jcmaster.c` | Compression master control; parameter validation, pass sequencing |
| `jccoefct.c` | Compression coefficient buffer controller (DCT → entropy) |
| `jcmainct.c` | Compression main buffer controller (preprocessor → coefficient compressor) |
| `jcprepct.c` | Compression preprocessing controller (input scanlines → downsampler) |
| `jccolor.c` | Compression colorspace conversion (RGB/CMYK/grayscale → YCbCr/YCCK) |
| `jcsample.c` | Compression downsampling (chroma subsampling) |
| `jcdctmgr.c` | Compression DCT manager; algorithm selection and quantization table setup |
| `jfdctflt.c` / `jfdctfst.c` / `jfdctint.c` | Forward DCT implementations (float, fast-integer, slow-integer) |
| `jchuff.c` | Sequential Huffman entropy encoder |
| `jcphuff.c` | Progressive Huffman entropy encoder |
| `jcmarker.c` | JPEG marker serializer (SOI, SOF, SOS, DHT, DQT, EOI, etc.) |
| `jcparam.c` | Compression parameter helpers (quantization tables, Huffman tables, defaults) |
| `jctrans.c` | Transcoding compression path (coefficients → output, bypassing pixel pipeline) |
| `jdapimin.c` / `jdapistd.c` | Minimum and standard decompression public API entry points |
| `jdinput.c` | Decompression input controller; marker/coefficient state machine |
| `jdmaster.c` | Decompression master control; module selection, output dimension calculation, pass management |
| `jdcoefct.c` | Decompression coefficient buffer controller (entropy decoder → IDCT) |
| `jdmainct.c` | Decompression main buffer controller (coefficient decoder → post-processor) |
| `jdpostct.c` | Decompression postprocessing controller (upsample → quantize) |
| `jdmarker.c` | JPEG marker parser; suspendable I/O, populates `j_decompress_ptr` fields |
| `jdcolor.c` | Decompression colorspace conversion (YCbCr/YCCK → RGB/CMYK/grayscale) |
| `jdsample.c` | Decompression upsampling (chroma expansion) |
| `jdmerge.c` | Merged upsampling + YCbCr→RGB for 2h1v/2h2v chroma subsampling (performance path) |
| `jddctmgr.c` | Decompression IDCT manager; algorithm selection and dequantization table setup |
| `jidctflt.c` / `jidctfst.c` / `jidctint.c` / `jidctred.c` | Inverse DCT implementations (float, fast-integer, slow-integer, reduced-size 4×4/2×2/1×1) |
| `jdhuff.c` | Sequential Huffman entropy decoder |
| `jdphuff.c` | Progressive Huffman entropy decoder |
| `jdtrans.c` | Transcoding decompression path (reads raw DCT coefficients without full decode) |
| `jquant1.c` | 1-pass color quantization (ordered / Floyd-Steinberg dithering) |
| `jquant2.c` | 2-pass color quantization (median-cut colormap + Floyd-Steinberg dithering) |
| `jmemmgr.c` | System-independent pool-based memory manager; virtual array management |
| `jmemnobs.c` | Quake renderer memory backend (no backing store) |
| `jmemansi.c` | ANSI stdlib memory backend with `tmpfile()` backing store |
| `jmemdos.c` | MS-DOS memory backend with XMS/EMS/file backing store |
| `jmemname.c` | Named-tempfile memory backend |
| `jpegtran.c` | Standalone lossless JPEG transcoder command-line tool |

---

## Core Responsibilities

- **JPEG decompression for texture loading**: decode JPEG files from the Quake virtual filesystem into raw RGBA/RGB pixel buffers consumed by the renderer.
- **Full JPEG compression pipeline**: colorspace conversion → downsampling → forward DCT → quantization → Huffman entropy coding → marker serialization → byte output.
- **Full JPEG decompression pipeline**: marker parsing → entropy decoding → inverse DCT → upsampling → colorspace conversion → optional color quantization → pixel output.
- **Multiple DCT algorithm variants**: float (`jfdctflt`/`jidctflt`), fast integer (`jfdctfst`/`jidctfst`), slow accurate integer (`jfdctint`/`jidctint`), and reduced-size IDCT (`jidctred`) selected per build configuration and output scaling.
- **Progressive and sequential JPEG support**: separate entropy codec modules (`jchuff`/`jcphuff`, `jdhuff`/`jdphuff`) handle both standard baseline and progressive scan coding.
- **Lossless transcoding path**: `jctrans.c`/`jdtrans.c` allow re-encoding a JPEG by operating directly on DCT coefficients, bypassing the pixel-domain pipeline.
- **Engine-integrated error and memory management**: `jerror.c` and `jmemnobs.c` replace all IJG standard I/O and allocation with renderer-native calls, preventing use of `stderr`, `malloc`, or backing store.

---

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `jload.c` — `LoadJPG(const char *filename, unsigned char **pic, int *width, int *height)` (inferred from responsibilities): the sole public surface consumed by the renderer's texture loader.
- All standard IJG API entry points (`jpeg_create_decompress`, `jpeg_read_header`, `jpeg_start_decompress`, `jpeg_read_scanlines`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, etc.) declared in `jpeglib.h`, available to any engine module that includes it.

**Consumed from other subsystems:**
- **Filesystem**: `FS_FOpenFileRead` / `FS_FCloseFile` — declared in `code/qcommon/qcommon.h`; used by `jload.c` to open JPEG image files from the virtual filesystem.
- **Zone allocator**: `Z_Malloc` — declared in `code/game/q_shared.h`; used by `jload.c` to allocate the output pixel buffer.
- **Renderer import table** (`refimport_t ri`): `ri.Error`, `ri.Printf` (used in `jerror.c`), `ri.Malloc`, `ri.Free` (used in `jmemnobs.c`) — provided by `code/renderer/tr_local.h` and populated at renderer initialization.

**Internal data flow (decompression):**

```
jdatasrc (in-memory buffer)
  → jdmarker (marker parsing, header decode)
  → jdinput (input state machine)
  → jdhuff / jdphuff (entropy decode)
  → jdcoefct (coefficient buffer)
  → jddctmgr → jidct* (inverse DCT + dequantize)
  → jdmainct (intermediate sample buffer)
  → jdsample / jdmerge (upsample)
  → jdcolor (colorspace conversion)
  → jdpostct → jquant1/jquant2 (optional color quantization)
  → caller output buffer
```

---

## Runtime Role

- **Init**: `jload.c` invokes `jpeg_create_decompress`, installs the Quake error manager (`jpeg_std_error` populated by `jerror.c`) and the in-memory source manager (`jdatasrc.c`), then calls `jpeg_read_header` to parse image metadata. The IJG memory manager (`jmemmgr.c` + `jmemnobs.c`) is initialized at `jpeg_create_decompress` time and all module vtables are wired by `jdinput.c` / `jdmaster.c` on the first `jpeg_start_decompress` call.
- **Frame**: JPEG loading is an on-demand, synchronous operation triggered during texture upload (not per-frame). `jpeg_read_scanlines` is called in a loop until all rows are consumed.
- **Shutdown**: `jpeg_finish_decompress` and `jpeg_destroy_decompress` release all pool-allocated memory via `jmemmgr.c`, which routes frees through `ri.Free`.

---

## Notable Implementation Details

- **Quake-specific error integration**: `jerror.c` is a non-standard replacement of the IJG default. Fatal JPEG errors call `ri.Error(ERR_FATAL, ...)`, making JPEG decode failures hard engine crashes rather than graceful exits.
- **No backing store**: `jmemnobs.c` unconditionally errors on any request for disk-based virtual array overflow. This means all JPEG working memory must fit in `ri.Malloc`-managed heap; very large images with progressive encoding and multi-pass quantization could theoretically trigger this error.
- **Memory-buffer source manager**: `jdatasrc.c` is described as modified from the stdio-original to read from an in-memory byte pointer rather than a `FILE*`. This matches the engine's filesystem model where file contents are loaded into a buffer before parsing.
- **Merged upsampling fast path**: `jdmerge.c` combines chroma upsampling and YCbCr→RGB conversion in a single pass for the 2h1v and 2h2v subsampling cases, avoiding redundant per-pixel multiplications.
- **Reduced-size IDCT**: `jidctred.c` produces 4×4, 2×2, or 1×1 output directly from an 8×8 DCT block, enabling native output scaling without a separate downsampling step.
- **Conditional DCT compilation**: forward and inverse DCT variants are individually guarded by `DCT_ISLOW_SUPPORTED`, `DCT_IFAST_SUPPORTED`, and `DCT_FLOAT_SUPPORTED` flags in `jconfig.h`, allowing dead DCT code to be excluded at build time.
- **Vendored library version**: the embedded library is IJG JPEG release 6, dated 1995 (`jversion.h`: `"6  2-Aug-95"`), predating libjpeg-6b and all subsequent releases; no security patches or API updates from later versions are present.
- **`jpegtran.c` is a standalone tool**: it is a command-line application entry point (`main()`), not linked into the engine binary. It exists in this directory as part of the original IJG source distribution.
