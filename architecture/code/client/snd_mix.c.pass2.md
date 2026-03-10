# code/client/snd_mix.c — Enhanced Analysis

## Architectural Role

This file implements the **per-frame audio mixing hub** that connects the client's high-level sound channel abstraction to the platform's low-level DMA output buffer. It sits between `snd_dma.c` (which manages channel lifecycle and DMA state) and platform-specific sound drivers (`SNDDMA_*`). Every frame, `S_PaintChannels` collects active one-shot and looping channels, mixes them into an intermediate `paintbuffer`, applies format/bit-depth conversions, and transfers the result to DMA memory for hardware playback.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/client/snd_dma.c`**: Calls `S_PaintChannels(endtime)` once per audio update tick; owns the `s_channels[]`, `loop_channels[]`, and global `dma` descriptor that this file reads/modifies
- **`code/client/cl_main.c`**: Indirectly via the client frame loop that drives `snd_dma.c`

### Outgoing (what this file depends on)
- **`snd_dma.c` globals**: `s_paintedtime`, `s_rawsamples`, `s_rawend`, `s_channels[]`, `loop_channels[]`, `dma` (output buffer descriptor), `s_testsound`, `s_volume` cvars
- **`snd_mem.c`**: `sfxScratchBuffer`, `sfxScratchPointer`, `sfxScratchIndex` (caches decoded chunks)
- **`snd_adpcm.c`**: `S_AdpcmGetSamples()`, `mulawToShort[256]` (Mu-Law decode table)
- **`snd_wavelet.c`**: `decodeWavelet()`
- **Platform DMA layer**: Writes directly to `dma.buffer` (physical or DMA-mapped memory)
- **Platform assembly** (Linux x86): `unix/snd_mixa.s` implements `S_WriteLinearBlastStereo16` for optimized stereo 16-bit transfer
- **`qcommon`**: `Com_Memset()` for clearing the paint buffer each iteration

## Design Patterns & Rationale

1. **Static intermediate buffer** — `paintbuffer[]` accumulates samples across all channels as 32-bit signed integers before transfer. This provides:
   - Headroom to prevent clipping when mixing multiple loud channels
   - Decoupling: hardware format changes don't affect channel mixing logic
   - Bounded memory: one 4096-sample buffer regardless of channel count

2. **Circular DMA wrap-around handling** — `S_TransferStereo16` explicitly breaks the transfer into linear segments using `lpos & mask` and advancing `snd_out`, avoiding a separate memcpy for wrap-around. Classic ring-buffer pattern from embedded systems.

3. **Compile-time polymorphism** — Three implementations of `S_WriteLinearBlastStereo16` (portable C, x86 naked asm, extern from `.s` file) selected at build time via `#if` guards. Allows platform-specific optimization (x86 asm uses `sar`, `or` to pack two 16-bit samples into one 32-bit write) without runtime cost.

4. **Format-specific mixing functions** — Four `S_PaintChannelFrom*` variants (16-bit PCM, ADPCM, Wavelet, Mu-Law) reflect that Quake III supported multiple audio encodings for different platforms/storage constraints.

5. **SIMD acceleration (AltiVec)** — The PCM path includes a vectorized loop processing 8 samples per iteration on PowerPC using `vec_mule`/`vec_mulo` (even/odd multiply-add pairs). Shows multi-architecture optimization typical of 2000s console ports.

6. **Doppler approximation via sample averaging** — Rather than true sinc resampling, the Doppler path averages samples across a fractional step interval (`aoff` to `boff`). Computationally cheap pitch shifting at the cost of quality.

7. **Decoder scratch buffer caching** — ADPCM/Wavelet paths cache decompressed output in `sfxScratchBuffer` with `sfxScratchIndex`/`sfxScratchPointer` to detect chunk boundaries and avoid re-decoding the same chunk in a single frame.

## Data Flow Through This File

```
S_PaintChannels(endtime)
  ├─ [Loop until s_paintedtime >= endtime in PAINTBUFFER_SIZE chunks]
  │   ├─ Com_Memset(paintbuffer, 0)                    [Clear accumulator]
  │   ├─ [Raw stream background: write s_rawsamples → paintbuffer]
  │   ├─ [For each one-shot channel: S_PaintChannelFrom{16,ADPCM,Wavelet,MuLaw}]
  │   │    └─ Read ch->doppler, ch->leftvol/rightvol, sfx format
  │   │    └─ Advance through sndBuffer linked list, decode/scale, accumulate
  │   ├─ [For each loop channel: same dispatch]
  │   └─ S_TransferPaintBuffer(endtime)                 [Pack to output format]
  │       ├─ S_TransferStereo16 [fast path: 16-bit stereo]
  │       │    └─ S_WriteLinearBlastStereo16 [clamping + bit-shift]
  │       └─ [Slow path: 8-bit mono/stereo via per-sample loop]
```

**32-bit paintbuffer → sample clamping ([-32768, 32767]) → 8/16-bit output → circular DMA buffer**

## Learning Notes

**Idiomatic to this era (2000s):**
- Global non-static variables (`snd_p`, `snd_out`, `snd_linear_count`) exposed for assembly code to use; would be a threading nightmare in modern code
- Compile-time CPU detection (`id386`, `idppc_altivec`) for platform-specific codepaths rather than runtime CPUID
- Explicit circular buffer management instead of relying on OS DMA abstractions
- Multiple compression formats hardcoded in as separate functions rather than a codec plugin system

**Contrasts with modern audio engines:**
- GPU-based mixing or at least SIMD-everywhere via libraries (XSIMD, Ipp) instead of ad-hoc AltiVec sections
- Streaming APIs (WASAPI, PulseAudio) handle circular buffering; no manual DMA ring-buffer management needed
- Format-agnostic mixing (decode to float once, mix, convert to output format) vs. separate per-format paths
- Thread-safe mixer with lock-free or message-passing patterns, not global state

**Key architectural insight:** This file is a **sampling rate / format adapter** — it normalizes all incoming sound (different compression, sample rates via Doppler) into one intermediate representation (32-bit stereo samples), mixes, and packs for hardware. Modern GPUs/APIs often do this implicitly, but Q3A's software mixing made it explicit and optimizable.

## Potential Issues

1. **Thread safety**: Global variables `snd_p`, `snd_out`, `snd_linear_count` shared with assembly code are not protected; if `S_PaintChannels` ever runs concurrently, data corruption is guaranteed.

2. **Doppler accumulation**: Floating-point `ooff` in the Doppler path can accumulate rounding error over long samples, potentially causing artifacts in heavily pitch-shifted sounds.

3. **Unsanitized chunk traversal**: The loops advancing through `sndBuffer` linked lists assume valid chains (e.g., `chunk = chunk->next`); a corrupted or circular chain would loop infinitely. No length validation.

4. **No volume overflow protection on raw samples**: `s_rawsamples` are mixed as-is without pre-scaling, potentially overflowing the accumulator if loud raw audio is mixed with channels at high volume.
