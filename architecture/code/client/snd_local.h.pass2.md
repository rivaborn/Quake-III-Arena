# code/client/snd_local.h — Enhanced Analysis

## Architectural Role

This header defines the **private mixing contract** between the client's sound subsystem (snd_dma.c, snd_mem.c, etc.) and the platform DMA layer (win_snd.c, linux_snd.c). It is not part of the public client API; rather, it unifies all internal sound state (mixing channels, listener pose, DMA ring buffer descriptor) and declares the platform-abstraction boundary (`SNDDMA_*` stubs). The file enforces a clean **software-mixing architecture** where all audio processing occurs on-CPU into a ring buffer that the platform layer streams to hardware.

## Key Cross-References

### Incoming (who depends on this file)
- All client-side sound implementation files (`snd_dma.c`, `snd_mem.c`, `snd_mix.c`, `snd_adpcm.c`, `snd_wavelet.c`) include and use the types/globals defined here
- Platform-layer DMA drivers (`win32/win_snd.c`, `unix/linux_snd.c`) implement the `SNDDMA_*` stubs declared here
- The renderer and game VMs do **not** directly reference this header; they call through the public `snd_public.h` API

### Outgoing (what this file depends on)
- Imports foundational types from `q_shared.h` (`vec3_t`, `qboolean`, `cvar_t`, `byte`, `MAX_QPATH`)
- Imports engine services from `qcommon.h` (memory allocation, filesystem, cvar management)
- Declares exports to `snd_public.h` (the public sound API contract visible to cgame/engine)
- Platform layer is the **client** of the `SNDDMA_*` interface; these functions are defined *elsewhere* per platform

## Design Patterns & Rationale

### Software Mixer + Ring Buffer
The `dma_t` descriptor and `s_rawsamples[MAX_RAW_SAMPLES]` implement a classic **real-time software audio mixer**. Unlike hardware-assisted mixing, *all* blend arithmetic happens on CPU: channels are mixed into `portable_samplepair_t` (32-bit stereo pairs for precision), then clamped and output. This choice trades CPU for simplicity and cross-platform portability—2005-era hardware mixing support was inconsistent.

### ADPCM State per Buffer
Each `sndBuffer` node embeds an `adpcm_state_t`, allowing **progressive decompression** during playback. Instead of decompressing entire sounds on load, ADPCM state carries the previous sample and step-table index; decode happens frame-by-frame. This amortizes CPU cost across playback.

### Pool Allocator for sndBuffer
The `SND_malloc/SND_free/SND_setup` functions prevent per-frame heap churn by pre-allocating a fixed `sndBuffer` pool. During the hot mixing loop, allocation is instant; eviction (`S_FreeOldestSound`) uses LRU timestamps on `sfx_t` to free memory when the pool is exhausted.

### Global Listener State
`listener_forward/right/up` are globals because Quake III assumes a **single listen point** per client. `S_Spatialize` uses these vectors to compute angle-of-arrival and apply L/R volume scaling + Doppler shift. (Split-screen would require per-viewpoint channel lists, not done here.)

### Cvar-Driven Configuration
Sound parameters (`s_volume`, `s_khz`, `s_mixahead`) are registered as cvars, allowing in-game tuning without recompile. This reflects Q3's design philosophy: all gameplay-relevant constants are exposed to `.cfg` scripting.

## Data Flow Through This File

**Load Path:**
1. `S_RegisterSound` (snd_dma.c) allocates `sfx_t`, sets `soundName`
2. `S_LoadSound` reads WAV file, parses `wavinfo_t`, allocates `sndBuffer` chain via `SND_malloc`
3. Optional encoding: `S_AdpcmEncodeSound`, `encodeWavelet`, `encodeMuLaw` compress into the buffer chain
4. `sfx->soundData` points to head of chain; `sfx->lastTimeUsed` set (LRU key)

**Play Path (per-frame):**
1. `S_Spatialize` reads entity origin (or `fixed_origin`), computes listener-relative angle → left/right volumes + Doppler scale
2. `S_PaintChannels(endtime)` iterates all active `s_channels[MAX_CHANNELS]` and `loop_channels[MAX_CHANNELS]`
3. For each channel: decompress current `sndBuffer` chunk via `S_AdpcmGetSamples` or `decodeWavelet`, apply spatialized volume, add to DMA paint buffer
4. Loop ends when painted time reaches hardware DMA cursor (via `SNDDMA_GetDMAPos`)
5. `SNDDMA_Submit` signals platform to begin playback; next frame syncs to new cursor position

**Memory Pressure:**
- If `SND_malloc` pool exhausted: `S_FreeOldestSound` finds the `sfx_t` with minimum `lastTimeUsed`, frees its entire `sndBuffer` chain, sets `inMemory = false`
- Sound reloads on next play request

## Learning Notes

### Era-Appropriate Tradeoffs
Q3's mixer is **purely software-based** because (1) DirectSound/ALSA mixing support was fragile in 2005, (2) CPU was cheap; power/latency were not primary concerns. Modern engines (Wwise, FMOD) use hardware mixers and provide plug-in DSP chains—Q3's approach would not scale to hundreds of simultaneous sounds.

### Doppler Without Frequency Modulation
The `dopplerScale` fields apply a *volume envelope* approximation to Doppler, not true frequency shifting. Real Doppler requires resampling the sound during playback. Q3's approach is CPU-efficient but perceptually subtle—listeners primarily notice left/right volume panning, not pitch shift.

### Single Listener Assumption
`listener_forward`, `listener_right`, `listener_up` are **not entity-relative**; they represent a fixed world-space orientation. This design assumes a single listening point per client—multiplayer split-screen or third-person listen-at-entity would require architectural changes (per-listener channel lists, dynamic basis vectors, etc.).

### Shared Spatialization Code
`S_Spatialize` is called per-channel per-frame, making it a hot path. The decision to compute angles in real-time (rather than pre-bake) reflects Q3's philosophy: dynamic entities and player movement mean spatialization must be frame-latest.

## Potential Issues

1. **Fixed Raw Buffer Size**: `s_rawsamples[MAX_RAW_SAMPLES]` (16384 stereo samples ≈ 372 ms at 44.1 kHz) is a hard limit. Excessive cinematic/voice streaming could overflow; no guard against wraparound is documented.

2. **Listener-Centric Only**: The global `listener_*` vectors cannot represent multiple independent listen points. Splitscreen or commander-cam would require refactoring to per-view channel allocation.

3. **No Thread Affinity Documented**: The mixing loop in `S_PaintChannels` is not marked thread-safe. If the renderer or server runs on another core (rare but possible), concurrent writes to `dma.buffer` could cause tearing. Platform layer should serialize via locks or provide double-buffering.

4. **Hardcoded Channel Limits**: `MAX_CHANNELS = 96` is split between one-shot and loop channels. If a level spawns many simultaneous sounds, older ones silently evict with no warning; no per-channel priority is documented.
