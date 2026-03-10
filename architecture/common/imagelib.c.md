# common/imagelib.c

## File Purpose
A build-tool/offline image I/O library (used by q3map, bspc, q3radiant, etc.) providing load and save routines for four legacy 2-D image formats: LBM (IFF-ILBM/PBM), PCX, BMP, and TGA. It is not part of the runtime engine; it runs on the host machine during asset processing.

## Core Responsibilities
- Read and decode LBM (PBM packed variant) files including RLE decompression
- Write LBM (PBM) files with FORM/BMHD/CMAP/BODY IFF chunks
- Read and decode PCX (ZSoft RLE) files
- Write PCX files (minimal RLE encoding)
- Read BMP files (BitmapInfo 40-byte and BitmapCore 12-byte headers, 8-bit only)
- Read TGA files (types 2/3/10, 24/32-bit) from file or in-memory buffer; write uncompressed 32-bit TGA
- Provide unified dispatch functions (`Load256Image`, `Save256Image`, `Load32BitImage`) that select the format by file extension

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bmhd_t` | struct | IFF BMHD chunk fields (dimensions, planes, masking, compression) |
| `pcx_t` | struct | PCX file header + inline pixel data marker |
| `TargaHeader` | struct | TGA file header (origin, dimensions, pixel depth, attributes) |
| `mask_t` | enum | LBM masking modes (ms_none, ms_mask, ms_transcolor, ms_lasso) |
| `compress_t` | enum | LBM compression modes (cm_none, cm_rle1) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `bmhd` | `bmhd_t` | global | Filled by `LoadLBM`; exposed externally so callers (e.g. `Load256Image`) can read image dimensions after load |

## Key Functions / Methods

### fgetLittleShort / fgetLittleLong
- **Signature:** `int fgetLittleShort(FILE *f)` / `int fgetLittleLong(FILE *f)`
- **Purpose:** Read little-endian 16/32-bit integers directly from a FILE stream.
- **Inputs:** Open FILE pointer.
- **Outputs/Return:** Decoded integer value.
- **Side effects:** Advances file position.
- **Calls:** `fgetc`
- **Notes:** Used only by `LoadBMP`; other loaders work from in-memory buffers.

### LBMRLEDecompress
- **Signature:** `byte *LBMRLEDecompress(byte *source, byte *unpacked, int bpwidth)`
- **Purpose:** Decode one scanline of IFF PackBits RLE into `unpacked`.
- **Inputs:** `source` — compressed data pointer; `unpacked` — destination buffer; `bpwidth` — expected decompressed byte count.
- **Outputs/Return:** Pointer to next compressed byte after this scanline.
- **Side effects:** Writes to `unpacked`. Calls `Error` if decompressed count exceeds `bpwidth`.
- **Calls:** `memset`, `memcpy`, `Error`
- **Notes:** `0x80` run-header is a NOP per IFF spec.

### LoadLBM
- **Signature:** `void LoadLBM(const char *filename, byte **picture, byte **palette)`
- **Purpose:** Parse a full IFF FORM/PBM file and return 8-bit pixel buffer and 768-byte palette.
- **Inputs:** Filename; pointers to receive picture and palette buffers (palette may be NULL).
- **Outputs/Return:** Allocates and sets `*picture`, optionally `*palette`; writes global `bmhd`.
- **Side effects:** `malloc` (picbuffer, cmapbuffer); `free` (LBMbuffer). Calls `Error` on format violations.
- **Calls:** `LoadFile`, `LBMRLEDecompress`, `memcpy`, `memset`, `malloc`, `free`, `Error`, `BigLong`, `BigShort`, `LittleLong`
- **Notes:** Only PBM (packed/chunky) is supported; ILBM (interleaved planar) is an `Error`. Chunk traversal uses `Align()` for even-byte padding.

### WriteLBMfile
- **Signature:** `void WriteLBMfile(const char *filename, byte *data, int width, int height, byte *palette)`
- **Purpose:** Encode and write a PBM IFF file with uncompressed BODY.
- **Inputs:** Destination filename, 8-bit pixel buffer, dimensions, 768-byte palette.
- **Outputs/Return:** None. Writes file to disk.
- **Side effects:** `malloc`/`free` for working buffer; calls `SaveFile`.
- **Calls:** `malloc`, `memset`, `memcpy`, `free`, `SaveFile`, `BigShort`, `BigLong`
- **Notes:** Writes big-endian IFF chunk lengths per spec.

### LoadPCX
- **Signature:** `void LoadPCX(const char *filename, byte **pic, byte **palette, int *width, int *height)`
- **Purpose:** Load and RLE-decode a PCX v5 8-bit image.
- **Inputs:** Filename; optional output pointers for pixels, palette, dimensions.
- **Outputs/Return:** Allocates `*pic` (if non-NULL) and `*palette` (if non-NULL).
- **Side effects:** `malloc`; calls `Error` on bad header or malformed data.
- **Calls:** `LoadFile`, `malloc`, `free`, `Error`, `LittleShort`
- **Notes:** Contains a `FIXME` comment about possible run-length overrun on the last row; truncates to avoid buffer overflow.

### WritePCXfile
- **Signature:** `void WritePCXfile(const char *filename, byte *data, int width, int height, byte *palette)`
- **Purpose:** Encode and write a PCX file; applies minimal RLE for pixels with high bits set.
- **Inputs:** Filename, raw 8-bit pixels, dimensions, 768-byte palette.
- **Side effects:** `malloc`/`free`; calls `SaveFile`.
- **Calls:** `malloc`, `memset`, `SaveFile`, `free`, `LittleShort`

### LoadBMP
- **Signature:** `void LoadBMP(const char *filename, byte **pic, byte **palette, int *width, int *height)`
- **Purpose:** Read an 8-bit Windows BMP (BitmapInfo or BitmapCore header variant).
- **Inputs:** Filename; optional output pointers.
- **Outputs/Return:** Allocates `*pic` and `*palette` (if non-NULL). Handles top-down (negative height) bitmaps by flipping rows.
- **Side effects:** File I/O via `fopen`/`fread`/`fseek`/`fclose`; `malloc`.
- **Calls:** `fopen`, `fgetLittleShort`, `fgetLittleLong`, `fseek`, `fread`, `fclose`, `malloc`, `Error`
- **Notes:** Converts BGR palette to RGB on output. Rejects non-8-bit and multi-plane images.

### LoadTGABuffer
- **Signature:** `void LoadTGABuffer(byte *buffer, byte **pic, int *width, int *height)`
- **Purpose:** Decode TGA from an in-memory buffer; supports types 2 (RGB), 3 (gray), 10 (RLE RGB), 24/32-bit.
- **Inputs:** In-memory byte buffer; output pointers.
- **Outputs/Return:** Allocates `*pic` as RGBA (4 bytes/pixel), bottom-up row order.
- **Side effects:** `malloc`. Uses `goto breakOut` to exit nested row/column loop for RLE spanning rows.
- **Calls:** `malloc`, `LittleShort`, `Error`
- **Notes:** Commented-out `free(buffer)` at end — caller retains ownership. Gray (type 3) expands to RGBA by replicating the single channel.

### LoadTGA
- **Signature:** `void LoadTGA(const char *name, byte **pixels, int *width, int *height)`
- **Purpose:** Thin wrapper — loads file into memory, delegates to `LoadTGABuffer`.
- **Calls:** `LoadFile`, `LoadTGABuffer`, `Error`

### WriteTGA
- **Signature:** `void WriteTGA(const char *filename, byte *data, int width, int height)`
- **Purpose:** Write an uncompressed 32-bit TGA with swapped BGR→RGB channel order.
- **Side effects:** `malloc`/`free`; `fopen`/`fwrite`/`fclose`.
- **Calls:** `malloc`, `memset`, `fopen`, `fwrite`, `fclose`, `free`

### Load256Image / Save256Image
- **Purpose:** Extension-dispatching wrappers for 8-bit formats (lbm/pcx/bmp load; lbm/pcx save).
- **Calls:** `ExtractFileExtension`, `Q_stricmp`, `LoadLBM`, `LoadPCX`, `LoadBMP`, `WriteLBMfile`, `WritePCXfile`, `Error`

### Load32BitImage
- **Purpose:** Extension-dispatching wrapper returning RGBA pixels; delegates to `LoadTGA` for `.tga`, else calls `Load256Image` and expands the palette index buffer to 32-bit RGBA.
- **Calls:** `ExtractFileExtension`, `Q_stricmp`, `LoadTGA`, `Load256Image`, `malloc`

## Control Flow Notes
This file has no frame/update loop involvement. It is a pure offline utility invoked during map compilation (`q3map`, `bspc`) and the level editor (`q3radiant`) when loading or saving texture/image assets. There is no shutdown path; all allocations are caller-managed (callers `free` returned buffers).

## External Dependencies
- `cmdlib.h` — `LoadFile`, `SaveFile`, `Error`, `ExtractFileExtension`, `Q_stricmp`, `BigShort`, `BigLong`, `LittleShort`, `LittleLong`, `qboolean`, `byte` — all defined in `common/cmdlib.c`
- `imagelib.h` — declares all public functions in this file
- Standard C: `stdio.h`, `stdlib.h` (via cmdlib), `string.h` (`memset`, `memcpy`)
