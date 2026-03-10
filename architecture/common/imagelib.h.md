# common/imagelib.h

## File Purpose
Public header declaring image I/O utility functions used by offline tools (map compiler, BSP tools, editor). It provides a unified interface for loading and saving paletted (8-bit) and true-color (32-bit) images across multiple formats.

## Core Responsibilities
- Declare loaders and writers for LBM (Deluxe Paint) format
- Declare loaders and writers for PCX (ZSoft) format
- Declare loaders and writers for TGA (Targa) format
- Provide format-agnostic wrappers (`Load256Image`, `Save256Image`) that dispatch by file extension
- Provide a unified 32-bit RGBA loader (`Load32BitImage`) abstracting format details

## Key Types / Data Structures
None. All data is passed as raw `byte *` / `unsigned *` pointers; no structs are declared here.

## Global / File-Static State
None.

## Key Functions / Methods

### LoadLBM
- Signature: `void LoadLBM(const char *filename, byte **picture, byte **palette)`
- Purpose: Load a Deluxe Paint LBM file into a heap-allocated pixel buffer and palette.
- Inputs: `filename` — path to file; `picture` — out-pointer for pixel data; `palette` — out-pointer for 256-entry RGB palette.
- Outputs/Return: Void; fills `*picture` and `*palette`.
- Side effects: Allocates heap memory for both buffers.
- Calls: Defined elsewhere (`common/imagelib.c` or equivalent).
- Notes: Caller is responsible for freeing returned buffers.

### WriteLBMfile
- Signature: `void WriteLBMfile(const char *filename, byte *data, int width, int height, byte *palette)`
- Purpose: Write an 8-bit paletted image to an LBM file on disk.
- Inputs: pixel buffer, dimensions, 256-entry palette.
- Outputs/Return: Void; writes file to disk.
- Side effects: File I/O.
- Calls: Defined elsewhere.
- Notes: None inferable.

### LoadPCX
- Signature: `void LoadPCX(const char *filename, byte **picture, byte **palette, int *width, int *height)`
- Purpose: Load a PCX image, returning pixel data, palette, and dimensions.
- Inputs: `filename`; out-pointers for pixels, palette, width, height.
- Outputs/Return: Void; fills all out-pointers.
- Side effects: Heap allocation for pixel and palette buffers.
- Calls: Defined elsewhere.
- Notes: None inferable.

### WritePCXfile
- Signature: `void WritePCXfile(const char *filename, byte *data, int width, int height, byte *palette)`
- Purpose: Write an 8-bit paletted image to a PCX file.
- Side effects: File I/O.

### Load256Image / Save256Image
- Signature: `void Load256Image(const char *name, byte **pixels, byte **palette, int *width, int *height)` / `void Save256Image(const char *name, byte *pixels, byte *palette, int width, int height)`
- Purpose: Format-agnostic 8-bit image load/save; dispatches to LBM or PCX handlers based on file extension.
- Inputs: `name` determines format via extension.
- Outputs/Return: Load fills pixel, palette, and dimension out-pointers; Save writes file.
- Side effects: Heap allocation on load; file I/O on save.
- Notes: Simplifies call sites that don't need to know the concrete format.

### LoadTGA
- Signature: `void LoadTGA(const char *filename, byte **pixels, int *width, int *height)`
- Purpose: Load a TGA file from disk into a heap-allocated 32-bit RGBA buffer.
- Side effects: Heap allocation; file I/O.

### LoadTGABuffer
- Signature: `void LoadTGABuffer(byte *buffer, byte **pic, int *width, int *height)`
- Purpose: Parse a TGA image from an already-resident memory buffer rather than from disk.
- Notes: Useful when the file has already been loaded into memory (e.g., from a pak).

### WriteTGA
- Signature: `void WriteTGA(const char *filename, byte *data, int width, int height)`
- Purpose: Write a 32-bit RGBA buffer to a TGA file on disk.
- Side effects: File I/O.

### Load32BitImage
- Signature: `void Load32BitImage(const char *name, unsigned **pixels, int *width, int *height)`
- Purpose: Format-agnostic 32-bit RGBA image loader; infers format from extension.
- Inputs: `name` — file path; out-pointers for pixel data and dimensions.
- Outputs/Return: Void; `*pixels` points to heap-allocated RGBA data.
- Side effects: Heap allocation.
- Notes: Higher-level companion to `Load256Image`; likely dispatches to `LoadTGA` or similar.

## Control Flow Notes
This is a pure header for offline build tools (`common/`, `q3map/`, `bspc/`, `q3radiant/`). It is not part of the runtime engine frame loop. Functions are called during asset loading at tool startup or on-demand when processing map/texture data.

## External Dependencies
- No includes declared in this header; consumers must include it after standard type headers (e.g., `cmdlib.h` for `byte`).
- All function bodies defined elsewhere (likely `common/imagelib.c`).
