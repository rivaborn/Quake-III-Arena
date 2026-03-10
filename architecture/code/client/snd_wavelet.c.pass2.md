# code/client/snd_wavelet.c — Enhanced Analysis

## Architectural Role

This file implements the audio compression/decompression layer bridging the sound asset loader (`snd_mem.c`) and the per-frame mixing engine (`snd_mix.c`). It provides dual compression strategies—wavelet-based (Daubechies-4 + mu-law) for smaller memory footprint, and mu-law-only with quantization dithering for lower CPU cost—as part of the Client subsystem's broader "software-mixed audio pipeline." Compressed audio lives in linked `sndBuffer` chunks attached to `sfx_t` sound assets; decompression happens on-demand during each frame's mixing loop.

## Key Cross-References

### Incoming (who depends on this file)
- `snd_mem.c` — calls `encodeWavelet()` and `encodeMuLaw()` during sound asset load
- `snd_mix.c` — calls `decodeWavelet()` and `decodeMuLaw()` per-frame during channel mixing
- Both codepaths access the `sndBuffer` chain anchored at `sfx->soundData` (type from `snd_local.h`)

### Outgoing (what this file depends on)
- `SND_malloc()` — defined in `snd_mem.c`; allocates fixed-size `sndBuffer` chunks from the sound heap
- `snd_local.h` — provides `sfx_t`, `sndBuffer`, `SND_CHUNK_SIZE`, `qboolean`, `byte`, `short`, `NXStream` typedefs
- Platform/math: `myftol()` declared but never called (vestigial)

## Design Patterns & Rationale

**Lazy-Initialized Lookup Table:** `mulawToShort[256]` is built once and guarded by `madeTable`, amortizing the cost of 256 `MuLawDecode()` calls across all sound playback sessions.

**In-Place Transforms:** Both `daub4()` and `wt1()` modify arrays in-place via workspace buffers, critical for memory-constrained 2005 era when stack was limited.

**Numerical Recipes Convention:** Internal use of 1-based indexing (`a = b - 1`) suggests code ported from Numerical Recipes or similar reference implementations; typical for academic-origin DSP code.

**Dual Compression Tradeoff:** 
- Wavelet path: smaller size but higher decompress cost (O(n log n) wavelet inverse each frame)
- Mu-law–only path: slightly larger, but O(n) lookup-table–based decompression with error feedback dithering to mask quantization noise

**Chunk-Based Streaming:** Fixed-size chunks (`SND_CHUNK_SIZE*2`, min 4) allow the loader to emit chunks progressively without loading entire sounds into contiguous memory—important for streaming large audio files.

## Data Flow Through This File

1. **Load phase** (`encodeWavelet` / `encodeMuLaw`): Raw 16-bit PCM samples flow from disk loader → chunked in 4–SND_CHUNK_SIZE*2 sample blocks → wavelet forward transform (if wavelet path) → mu-law quantization (8-bit) → linked `sndBuffer` chunks
2. **Play phase** (`decodeWavelet` / `decodeMuLaw`): Each frame, mixing engine iterates `sfx->soundData` chain → reads compressed `byte` samples → decompresses via lookup table and/or wavelet inverse → 16-bit PCM output → fed to mix/DMA layer
3. **Initialization guard**: First call to encode/decode ensures `mulawToShort[256]` table exists

## Learning Notes

This file exemplifies **early 2000s embedded audio DSP:**
- **Wavelet compression** (Daubechies-4): multiresolution decomposition was cutting-edge for console/streaming audio before ADPCM and MP3 became dominant; still used in some offline mastering pipelines
- **Mu-law encoding**: logarithmic quantization exploits human auditory masking; standard in telephony; 8-bit gives ~48 dB SNR, sufficient for speech/ambient game audio at 11–22 kHz
- **Error feedback dithering** in `encodeMuLaw()`: the `grade` accumulator reduces banding artifacts in low-amplitude signals—a key technique in audio quantization
- **Memory-first design**: No dynamic allocation in codec paths; all buffers preallocated on stack or via `SND_malloc()`

Modern engines instead:
- Precompress to MP3/OGG offline; load as opaque blobs
- Rely on hardware ADPCM (mobile) or lossless codecs (PC/console)
- Dynamically decompress only active channels, not on-asset-load

## Potential Issues

1. **Buffer overflow in `daub4()`** (line 39): Stack workspace `wksp[4097]` can overflow if `n > 4096`. The function does not validate input; correctness depends entirely on caller respecting the limit via `SND_CHUNK_SIZE*2` cap. If `SND_CHUNK_SIZE` is ever redefined ≥ 2049, overflow occurs.

2. **Possible out-of-bounds read in `encodeWavelet()`** (line 147): The `if (size < 4) { size = 4; }` forces minimum chunk size even if fewer samples remain. On the final chunk, if `sfx->soundLength % SND_CHUNK_SIZE*2 < 4`, the loop reads 4 samples from `packets` pointer but fewer are available. This likely works in practice because Q3A sounds are always padded, but is not validated.

3. **No bounds check on `decodeWavelet`/`decodeMuLaw`** (lines 173–174, 216): Both functions read `chunk->size` samples but never validate it does not exceed the actual `sndChunk` buffer capacity. If `size` is corrupted or a chunk is truncated, reads can overrun the buffer.

4. **Vestigial API:** `NXPutc()` and `NXStreamCount` (lines 122–125) are defined but never called in this file's encode/decode paths; likely remnants of an alternative output strategy.
