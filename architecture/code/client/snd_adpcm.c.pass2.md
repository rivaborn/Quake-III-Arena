# code/client/snd_adpcm.c — Enhanced Analysis

## Architectural Role
This file is one of several compression codecs in Quake III's software-mixed audio pipeline (alongside `snd_wavelet.c` and likely mu-law elsewhere in the sound system). It bridges the load-time asset pipeline (`S_AdpcmEncodeSound`) with the per-frame mix-time streaming path (`S_AdpcmGetSamples`). The chunked architecture with per-block state headers enables efficient streaming of large sounds through the fixed `PAINTBUFFER_SIZE` paint buffer without requiring decompressed-in-memory copies—critical for memory-constrained platforms of the era.

## Key Cross-References

### Incoming (who depends on this file)
- **`snd_mix.c`** or sound mix loop — calls `S_AdpcmGetSamples(chunk, dest)` during the per-frame paint path to decode chunks into the mix buffer
- **Asset loader** (likely in `snd_mem.c` or `cl_parse.c`) — calls `S_AdpcmEncodeSound()` when loading `.wav` files marked for ADPCM compression (load-time encoding)
- **Memory estimation** — `S_AdpcmMemoryNeeded()` is called by the sound allocator to reserve space before encoding

### Outgoing (what this file depends on)
- **`snd_mem.c`** — provides `SND_malloc()` allocator for `sndBuffer` chunks
- **`snd_local.h`** — defines `sndBuffer`, `sfx_t`, `adpcm_state_t` types; provides `PAINTBUFFER_SIZE` (likely 4096) and `SND_CHUNK_SIZE_BYTE` macros
- **Global `dma` object** (`dma_t` from client core) — `dma.speed` read by `S_AdpcmMemoryNeeded()` for sample rate scaling
- **`q_shared.h`** — basic types via `snd_local.h`

## Design Patterns & Rationale

**Per-Block State Headers:** Each `sndBuffer` chunk stores an `adpcm_state_t` header (last predicted sample + step index). This allows:
- **Streaming without random access:** Decoders can start from any block independently during long playback
- **Efficient seeking:** Jump to a block's header and decode forward without rewinding to file start
- **Deterministic memory layout:** State is embedded in the chunk structure, avoiding external state tables

**Chunked Pipeline:** The architecture chunks at `PAINTBUFFER_SIZE` (≈4096 samples), which matches the engine's per-frame mix buffer. This allows:
- **Single-pass decode-and-mix:** Paint loop decodes one chunk into the mix buffer in-place per frame
- **Fixed memory footprint:** Only one or two chunks in flight at a time, not the entire sound asset
- **Rate-scaled allocation:** `S_AdpcmMemoryNeeded()` pre-calculates for the playback rate, not the source rate

**Bit-Shift Approximation (not multiply/divide):** The encoder uses left/right shifts instead of hardware division (`delta = diff*4/step` → shift-based approximation with careful rounding). This was essential on 1990s–2000s CPUs where integer division was slow and multiplication had long latencies; modern CPUs would prefer explicit multiply-add sequences.

## Data Flow Through This File

```
LOAD TIME:
  WAV file (16-bit PCM)
       ↓ (passed to S_AdpcmEncodeSound)
  Encode loop: split into chunks of SND_CHUNK_SIZE_BYTE*2 samples
       ↓ (each chunk)
  S_AdpcmEncode (state-preserving)
       ↓ Output: 4-bit nibbles (2 per byte) in sndBuffer.sndChunk
  Linked list of sndBuffer nodes attached to sfx_t.soundData
       ↓
  MEMORY: (sampleMemory = scaledSampleCount/2) + (blockCount * sizeof(adpcm_state_t))

MIX TIME (per frame):
  For each playing sound:
    Current chunk (sndBuffer)
       ↓ (passed to S_AdpcmGetSamples)
    Read embedded adpcm_state_t header (initial state for this block)
    S_AdpcmDecode (stateless per-chunk call, local state var)
       ↓ Output: 16-bit PCM samples for SND_CHUNK_SIZE_BYTE*2 samples
    Paint buffer (short array)
       ↓ (continue to next chunk or finish)
  Advance to next sndBuffer node in linked list
```

**Rate scaling:** If source is 22 kHz and playback is 44 kHz, `S_AdpcmMemoryNeeded()` divides scaledSampleCount by 2, allocating half the bytes. The actual resampling likely happens elsewhere in the mix pipeline (not shown here).

## Learning Notes

**Codec era idiom:** IMA ADPCM (Intel/DVI) is a 1980s–1990s standard for real-time audio compression on limited hardware. Modern engines use Opus, Vorbis, or proprietary neural codecs; Q3A's choice reflects its contemporary constraints (memory, CPU, no floating-point audio preferred).

**Stateful codec design:** The codec must maintain `(sample, index)` state across block boundaries. The per-block header save is a clever workaround: instead of storing a monolithic state object between encode/decode calls, each block becomes independently decodable from its own header. This pattern appears in video codecs (I-frames) and modern streaming codecs.

**Determinism requirement:** Both encoder and decoder must follow identical arithmetic (the approximation in step 2's comment is critical). Any divergence causes drift. This is why the code comments explicitly note the shift-based approximation vs. true multiply/divide.

**Hardware audio interface:** The `dma` global and `PAINTBUFFER_SIZE` tie this to the client's DMA ring-buffer architecture (likely a platform-specific sound driver callback). The codec is sandwiched between asset decompression and platform-specific hardware mixing.

**Missing: compression ratio / quality tradeoffs**
- ADPCM achieves ~4:1 compression (16 bits → 4 bits) with acceptable quality for game sound effects
- Speech/music may require wavelet or mu-law (mentioned in architecture notes as alternatives)
- No adaptive bit-rate or quality settings visible here

## Potential Issues

1. **Odd sample handling in `S_AdpcmEncode`:** If `len` is odd, the final nibble is written as the high nibble of a partial byte (`outputbuffer`), then flushed. But the caller (`S_AdpcmEncodeSound`) always passes `n = min(count, SND_CHUNK_SIZE_BYTE*2)`, which is even. Risk is low but undocumented.

2. **No input validation:** `S_AdpcmMemoryNeeded` reads `dma.speed` without checking if DMA is initialized; could return garbage if called too early in startup.

3. **Float division rounding:** `scale = (float)info->rate / dma.speed;` then `scaledSampleCount = info->samples / scale;` can lose precision. A single integer division might be safer: `scaledSampleCount = (info->samples * dma.speed) / info->rate`.

4. **No bounds checking on `len` in decode:** If caller passes `len > SND_CHUNK_SIZE_BYTE*2`, the output buffer `outdata` overflows. Mitigated by protocol (paint loop always passes exact chunk size), but fragile.

5. **Global state potential:** `stepsizeTable[89]` and `indexTable[16]` are standard per spec, but no version/validation check if a future platform uses a different dialect.
