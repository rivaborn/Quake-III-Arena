# code/jpeg-6/jchuff.h — Enhanced Analysis

## Architectural Role
This header declares Huffman entropy encoder infrastructure shared between sequential and progressive JPEG encoder implementations within the vendored IJG libjpeg-6 library. While the entire JPEG library is integrated for runtime texture loading (via `tr_image.c` → `jload.c` decode pipeline), this specific encoder component is likely **dead code at runtime**—the renderer only needs JPEG *decoding*, not encoding. The header serves as part of a complete, vendored, self-contained library unit isolated from the rest of the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **jchuff.c** and **jcphuff.c** — Internal JPEG encoder modules that include this header and implement the encoder functions declared here
- No engine subsystems depend on encoder functions; texture loading uses only the decoder path

### Outgoing (what this file depends on)
- **jpeglib.h**, **jpegint.h** — JPEG library core infrastructure defining `j_compress_ptr`, `JHUFF_TBL`, memory manager, and macro wrappers (`JPP`, `EXTERN`)
- **libc** (indirectly via JPEG macros) for memory allocation through JPEG's memory manager

## Design Patterns & Rationale
**Pre-computation for performance**: The two-stage Huffman pipeline (generate optimal table from frequencies, then expand to derived form) reflects mid-1990s optimization: Huffman tables are computed once during encoder initialization, then every symbol encoding uses O(1) table lookup rather than tree traversal. The `c_derived_tbl` struct packs codes and lengths tightly (unsigned int + char per symbol × 256).

**Modularity across encoder variants**: Sharing declarations between sequential (`jchuff.c`) and progressive (`jcphuff.c`) encoders avoids duplication while allowing independent implementation details; both need the same pre-computed table format.

## Data Flow Through This File
1. **Input**: Raw frequency counts `freq[]` (symbol occurrence statistics)
2. **Stage 1** (`jpeg_gen_optimal_table`): Huffman tree construction → fills `JHUFF_TBL` with code lengths
3. **Stage 2** (`jpeg_make_c_derived_tbl`): Expands `JHUFF_TBL` → pre-computes codes and lengths in `c_derived_tbl`
4. **Output**: Fast lookup table ready for encoding pass (not exercised in typical renderer usage)

## Learning Notes
**Legacy compression strategy**: Represents a stable, well-understood JPEG encoder from the pre-2000 era. Modern game engines optimize texture compression differently (BCn/DXT for GPU cache, ZSTD for streaming), but Quake III keeps the full, vendored IJG library for compatibility and simplicity—paying the cost of unused encoder code in the binary.

**256-entry barrier**: The array sizes hardcoded to 256 symbols reflect JPEG's byte-value limitation; all symbols map to `[0..255]`. Contrast with modern variable-length codes over arbitrary alphabets.

## Potential Issues
- **Likely dead code**: No callchain from the renderer into `jpeg_make_c_derived_tbl` or `jpeg_gen_optimal_table` is inferable from the architecture; texture loading is decode-only.
- **No cross-boundary bounds checking** between `ehufco` and `ehufsi` arrays—caller must ensure consistency.
- **Symbol code safety**: A zero `ehufsi[S]` correctly signals "no code assigned," but callers must check before indexing `ehufco[S]`.
