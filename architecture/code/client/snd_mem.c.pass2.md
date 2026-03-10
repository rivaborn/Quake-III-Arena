# code/client/snd_mem.c — Enhanced Analysis

## Architectural Role

This file implements the **sound asset memory management and loading subsystem** for the Quake III client, acting as the interface between the virtual filesystem (WAV on disk) and the real-time mixing pipeline. It occupies a critical position in the **Client subsystem** (`code/client/`), sitting between the demand-driven asset loader and the per-frame DMA mixer (`snd_dma.c`, `snd_mix.c`). All loaded sounds flow through its allocator; eviction policy and mixing state are owned by `snd_dma.c`, but chunk lifetime and resampling are owned here. It is never called from the server or game VM.

## Key Cross-References

### Incoming (who depends on this file)

- **`SND_setup()`** ← called once from `S_Init()` in `snd_dma.c` (sound system startup)
- **`S_LoadSound()`** ← called on-demand from sound reference resolution (not part of per-frame loop; blocking I/O)
- **`SND_malloc()` / `SND_free()`** ← called throughout the mixer (`snd_mix.c`) when fetching/releasing chunks; also called during `S_LoadSound` resampling
- **Global `sfxScratchBuffer`, `sfxScratchPointer`, `sfxScratchIndex`** ← read/written by mixing and compression code (`snd_mix.c`, `snd_adpcm.c`, `snd_wavelet.c`) during playback and encoding

### Outgoing (what this file depends on)

- **`dma` global** (`snd_dma_t`) from `snd_dma.c` — reads `dma.speed` for resampling ratio; defines the target sample rate
- **`S_FreeOldestSound()`** from `snd_dma.c` — eviction callback; implements LRU policy when allocator is exhausted
- **`S_AdpcmEncodeSound()`** from `snd_adpcm.c` — optional post-load compression (16-bit → ADPCM); called if `sfx->soundCompressed == qtrue`
- **Engine common layer** (`qcommon/`): `FS_ReadFile`, `FS_FreeFile`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Com_Milliseconds`, `Cvar_Get`, `Com_Printf`, `Com_DPrintf`, `Com_Memset`
- **Math/utility** from `q_shared.h`: `LittleShort` macro (endian swap)

## Design Patterns & Rationale

**1. Fixed-Size Pool + Free-List Allocator**
- Allocates a contiguous slab of `sndBuffer` chunks at startup (size = `com_soundMegs * 1536` chunks)
- Builds a linked-list free-list in reverse order through the slab
- Why: Pre-allocation prevents runtime fragmentation and GC pauses during gameplay; fixed chunk size (1024 shorts) matches DMA constraints
- Evicts oldest sound on exhaustion (not FIFO; LRU via `sfx->lastTimeUsed`)

**2. Stateful RIFF Parser**
- Uses file-static cursor variables (`data_p`, `last_chunk`, `iff_data`, etc.) to traverse RIFF chunks sequentially
- Why: Minimizes stack usage; leverages the sequential nature of RIFF container format
- Not thread-safe, but single-threaded engine assumption holds

**3. Scratch Buffer Pattern**
- `sfxScratchBuffer` is a shared temporary (pre-allocated once in `SND_setup`)
- During load: raw PCM → resample/decompress → scratch buffer → optionally compress → final sndBuffer chain
- Why: Avoids allocating a temporary for every sound load; single 4× buffer (4 ints per sample for filtering headroom) suffices

**4. On-Demand Loading with Blocking I/O**
- No preloading; `S_LoadSound` is called when a sound is first referenced (lazy initialization)
- Calls `FS_ReadFile` (blocking), then synchronously resamples
- Why: Saves startup time and memory; Q3's gameplay is network-bound anyway (latency dominates audio load time)

**5. Optional Compression Pipeline**
- Uncompressed path is primary (line 330: `sfx->soundCompressionMethod = 0`)
- ADPCM encoding path is active; Mu-law/Wavelet paths are compiled out (`#if 0`)
- Why: ADPCM reduces memory footprint by ~4×; uncompressed path is fallback for audio quality

## Data Flow Through This File

1. **Initialization** (`SND_setup`)
   - Allocate slab: `buffer` (sized by `com_soundMegs` cvar, default 8 MB → 12,288 chunks)
   - Allocate scratch: `sfxScratchBuffer` (1024 × 4 shorts = 8 KB)
   - Initialize free-list: link all chunks in reverse, head at `freelist`

2. **Sound Load** (`S_LoadSound` → `GetWavinfo` → `ResampleSfx`)
   ```
   WAV file (disk) 
      ↓ FS_ReadFile
   Raw bytes in memory 
      ↓ GetWavinfo (parse RIFF/fmt/data chunks)
   wavinfo_t (format metadata)
      ↓ Hunk_AllocateTempMemory (scratch space)
   ResampleSfx or ResampleSfxRaw (fractional-rate resampling)
      ↓ SND_malloc × N (allocate linked sndBuffer chunks)
   sfx->soundData chain (final resident form)
      ↓ [optional] S_AdpcmEncodeSound (compress in-place)
   Compressed sfx or raw PCM chain
   ```

3. **Playback** (not in this file, but `snd_mix.c` reads)
   - Mixer traverses `sfx->soundData` linked list
   - Reads samples from `chunk->sndChunk[sample_index]`
   - Refills `sfxScratchBuffer` on chunk boundary (for ADPCM state alignment)

4. **Eviction** (on memory pressure)
   - `SND_malloc` → `S_FreeOldestSound` → `SND_free` (return chunks to freelist)

## Learning Notes

- **Era-appropriate resampling**: Linear interpolation (`samplefrac >> 8` + fractional step) is simple and sufficient for 1990s audio; modern engines use sinc or polyphase kernels
- **Idiomatic: WAV-only input**: No Ogg/MP3 support. Q3's file sizes were designed around uncompressed PCM + optional ADPCM; mod community later added Ogg via engine patches
- **Design philosophy**: **Pre-allocate, never free until shutdown**. Sound chunks are not returned to OS; eviction is within the fixed pool
- **Mono-only assumption**: Stereo rejection (line 281) enforces single-channel for simplicity; stereo would require 2× bandwidth
- **22 kHz target**: Non-22 kHz sources are accepted but trigger a warning (line 287). This was the runtime target rate for Q3 (CD audio quality at low CPU cost in 1999)
- **Scratch buffer multiplexing**: `sfxScratchPointer` tracks ownership to prevent overwrite during concurrent operations (not visible in this file, but used by `snd_mix.c`)

## Potential Issues

1. **Buffer overflow in `ResampleSfx`** (line 246–266): If `SND_malloc()` returns NULL (freelist empty, eviction fails), the subsequent write to `chunk->sndChunk[part]` is undefined. No null-check or error handling.

2. **WAV parser state corruption**: File-static cursor variables (`data_p`, `iff_data`, `last_chunk`) are not reset between calls. If `FindChunk` is called twice on different WAV files, the second call may read stale `last_chunk` offsets. (Mitigated in practice because `GetWavinfo` always calls `FindChunk("RIFF")` first, which resets via `last_chunk = iff_data`.)

3. **Integer division by zero** (line 214): `info.width = GetLittleShort() / 8;` — if width field in WAV header is 0, info.width becomes 0, then line 330 divides by zero in sample count calculation. No validation.

4. **Scratch buffer race** (cross-file): `sfxScratchBuffer` is shared global; if load and mixing happen concurrently (not true in Q3, but a code smell), a second `S_LoadSound` could corrupt in-progress compression. Mitigated by single-threaded guarantee.
