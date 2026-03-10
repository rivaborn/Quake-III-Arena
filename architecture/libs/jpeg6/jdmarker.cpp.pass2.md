# libs/jpeg6/jdmarker.cpp — Enhanced Analysis

## Architectural Role

This file is the JPEG marker decoder from the vendored IJG libjpeg-6 library. Located in `libs/jpeg6/` (tool utilities), it mirrors `code/jpeg-6/` (runtime). In both contexts, it parses JPEG segment markers and extracts image metadata. The renderer (`code/renderer/tr_image.c`) depends on this to load JPEG textures; offline tools in `q3map/` and `q3radiant/` also use it for image asset loading.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer** (`code/renderer/tr_image.c`): Calls decompressor API to load JPEG files as GPU textures
- **Offline tools** (`q3map/`, `q3radiant/`): Use the same libjpeg routines for asset import and image manipulation
- **jload.c** (same library): Wraps this decompressor into a higher-level image-loading interface

### Outgoing (what this file depends on)
- **jpeglib.h**, **jinclude.h**: Standard JPEG library headers
- **Memory allocator** (`cinfo->mem->alloc_small`): Delegated to host engine (qcommon hunk allocator in runtime, tool allocator in tools)
- **Input source manager** (`cinfo->src->fill_input_buffer`, `skip_input_data`): Supplied by caller; abstracts file I/O and buffering
- **Huffman/arithmetic subsystems**: Called after this function populates tables; decoded in `jdh*.c` and `jda*.c` siblings

## Design Patterns & Rationale

**Marker-driven dispatch**: Each segment type (SOI, SOF, SOS, DQT, DHT, APP0, APP14, etc.) has its own handler (`get_soi`, `get_sof`, etc.). This modular design isolates parsing logic and makes the decoder resilient to unknown marker extensions.

**Input suspension protocol**: The `INPUT_VARS`, `INPUT_SYNC`, `INPUT_RELOAD` macros orchestrate a careful dance with the external input source manager. If a marker's data isn't fully available, the function returns FALSE and the decoder suspends—then resumes the *same* function on the next data arrival. This allows predictable memory usage and true streaming without buffering entire files.

**Bi-sync local copies**: The decoder maintains local shadow copies of `next_input_byte` and `bytes_in_buffer` from `cinfo->src`, syncing them only at restart boundaries. This prevents partial state corruption if suspension interrupts a marker parse mid-stream.

## Data Flow Through This File

**Input:** Raw JPEG bitstream; marker code already decoded and stored in `cinfo->unread_marker`

**Processing:**
1. Dispatch to handler based on marker code (SOI, SOF0–15, DHT, DQT, DRI, DAC, APP0–15, COM, SOS, EOI)
2. Read segment length and validate it against expected parameter counts
3. Parse segment-specific data: image dimensions, component sampling factors, quantization/Huffman/arithmetic table definitions, JFIF/Adobe metadata
4. Validate constraints (image height/width > 0, component counts, table indices in range)
5. Allocate `cinfo->comp_info` array once; populate component-specific state
6. Update marker-visit flags (`saw_SOI`, `saw_SOF`, `saw_SOS`) for ordering enforcement

**Output:** Fully populated `jpeg_decompress_struct`; return TRUE if complete, FALSE if suspended mid-marker

## Learning Notes

- **Idiomatic 1990s streaming**: The suspension/resumption protocol predates async I/O and is designed for embedded and streaming contexts demanding predictable, bounded memory. Modern engines use buffering; here, the caller supplies on-demand data arrival.
- **JPEG spec rigor**: Marker handlers validate segment lengths, component counts, table indices, and presence ordering (`JERR_SOS_NO_SOF`, `JERR_SOF_DUPLICATE`), ensuring strict spec compliance rather than best-effort parsing.
- **No C++ semantics**: Despite `.cpp` extension, the code is pure C (no OOP, no overloading). The extension is a Windows/MSVC build artifact.
- **Marker interleaving**: Enforces precise sequencing—DHT/DQT/DAC must precede SOS; SOI must precede all others—preventing partially initialized decompression.

## Potential Issues

None clearly inferable from the code. The implementation is defensive (bounds-checked, well-validated). However, the suspension protocol is subtle—any future modifications to marker parsing must be tested both with and without input suspension (simulated by partial buffer fills) to avoid state-sync bugs.
