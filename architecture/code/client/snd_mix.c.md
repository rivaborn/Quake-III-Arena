# code/client/snd_mix.c

## File Purpose
Implements the portable audio mixing pipeline for Quake III Arena's DMA sound system. It reads from active sound channels, mixes them into an intermediate paint buffer, and transfers the result into the platform DMA output buffer.

## Core Responsibilities
- Maintain and fill the stereo `paintbuffer` intermediate mix buffer
- Mix one-shot and looping sound channels into the paint buffer per-frame
- Support four audio decompression paths: raw PCM 16-bit, ADPCM, Wavelet, and Mu-Law
- Apply volume scaling and optional Doppler pitch shifting during mixing
- Transfer the paint buffer to the DMA output buffer with bit-depth/channel-count adaptation
- Provide platform-specific fast paths: x86 inline asm (`id386`) and AltiVec SIMD (`idppc_altivec`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `portable_samplepair_t` | struct | Stereo 32-bit integer sample pair used in the paint buffer (defined in `snd_local.h`) |
| `channel_t` | struct | Active sound channel with volume, position, Doppler state, and sfx pointer |
| `sfx_t` | struct | Sound effect descriptor including compression method, length, and linked chunk list |
| `sndBuffer` | struct | Linked-list node holding a chunk of decoded/raw audio samples |
| `dma_t` | struct | DMA output buffer descriptor: sample rate, bit depth, channel count, byte buffer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `paintbuffer` | `portable_samplepair_t[PAINTBUFFER_SIZE]` | static | Intermediate stereo 32-bit mix buffer (4096 sample pairs) |
| `snd_vol` | `int` | static | Scaled master volume (`s_volume * 255`), set each `S_PaintChannels` call |
| `snd_p` | `int *` | global (non-static) | Pointer into `paintbuffer`; shared with `unix/snd_mixa.s` |
| `snd_linear_count` | `int` | global | Sample count for the current linear blast; shared with asm |
| `snd_out` | `short *` | global | Output pointer into DMA buffer; shared with asm |

## Key Functions / Methods

### S_WriteLinearBlastStereo16
- **Signature:** `void S_WriteLinearBlastStereo16(void)`
- **Purpose:** Converts `snd_linear_count` 32-bit paint buffer samples to clamped 16-bit stereo output in `snd_out`.
- **Inputs:** Implicit — `snd_p`, `snd_linear_count`, `snd_out` globals.
- **Outputs/Return:** Writes to `snd_out` buffer (DMA memory).
- **Side effects:** Writes DMA output buffer region.
- **Calls:** None (pure data transform).
- **Notes:** Three implementations selected at compile time — portable C, x86 naked inline asm (`id386`), or extern forward declaration for Linux/FreeBSD x86 (implemented in `snd_mixa.s`). Clamps values to [-32768, 32767] and right-shifts by 8.

### S_TransferStereo16
- **Signature:** `void S_TransferStereo16(unsigned long *pbuf, int endtime)`
- **Purpose:** Loops over the circular DMA buffer, calling `S_WriteLinearBlastStereo16` in linear segments to handle wrap-around.
- **Inputs:** `pbuf` — DMA output buffer pointer; `endtime` — target sample time.
- **Outputs/Return:** None.
- **Side effects:** Sets `snd_p`, `snd_out`, `snd_linear_count`; writes to DMA buffer.
- **Calls:** `S_WriteLinearBlastStereo16`

### S_TransferPaintBuffer
- **Signature:** `void S_TransferPaintBuffer(int endtime)`
- **Purpose:** Transfers the completed paint buffer to the DMA byte buffer, handling all format combinations (16/8-bit, 1/2 channels). Optionally overwrites with a sine test tone.
- **Inputs:** `endtime` — upper time bound.
- **Side effects:** Writes `dma.buffer`; reads `s_testsound`, `dma`, `s_paintedtime`.
- **Calls:** `S_TransferStereo16`

### S_PaintChannelFrom16
- **Signature:** `static void S_PaintChannelFrom16(channel_t *ch, const sfx_t *sc, int count, int sampleOffset, int bufferOffset)`
- **Purpose:** Mixes raw 16-bit PCM sound data from a channel into the paint buffer with volume scaling. Handles Doppler resampling and AltiVec SIMD acceleration.
- **Inputs:** Channel, sfx, sample count, source offset, destination paint buffer offset.
- **Outputs/Return:** Accumulates into `paintbuffer`.
- **Side effects:** Reads/advances through `sndBuffer` linked list.
- **Notes:** AltiVec path processes 8 samples per vector loop iteration; Doppler path performs fractional pitch shift by averaging samples across the scaled step interval.

### S_PaintChannelFromADPCM
- **Signature:** `void S_PaintChannelFromADPCM(channel_t *ch, sfx_t *sc, int count, int sampleOffset, int bufferOffset)`
- **Purpose:** Mixes ADPCM-compressed audio by decoding chunks into `sfxScratchBuffer` on demand, then accumulating into the paint buffer.
- **Side effects:** May call `S_AdpcmGetSamples`; updates `sfxScratchIndex`, `sfxScratchPointer`.

### S_PaintChannelFromWavelet
- **Signature:** `void S_PaintChannelFromWavelet(channel_t *ch, sfx_t *sc, int count, int sampleOffset, int bufferOffset)`
- **Purpose:** Same role as ADPCM variant but for Wavelet-compressed audio; calls `decodeWavelet` on chunk boundaries.
- **Side effects:** Updates `sfxScratchIndex`, `sfxScratchPointer`; calls `S_AdpcmGetSamples`, `decodeWavelet`.

### S_PaintChannelFromMuLaw
- **Signature:** `void S_PaintChannelFromMuLaw(channel_t *ch, sfx_t *sc, int count, int sampleOffset, int bufferOffset)`
- **Purpose:** Mixes Mu-Law encoded audio by table-lookup (`mulawToShort`) with optional Doppler. Supports looping across chunk boundaries.

### S_PaintChannels
- **Signature:** `void S_PaintChannels(int endtime)`
- **Purpose:** Main per-frame mixing entry point. Fills the paint buffer in PAINTBUFFER_SIZE chunks, blending streaming background audio (`s_rawsamples`), one-shot channels, and looping channels, then transfers each completed chunk to the DMA buffer.
- **Inputs:** `endtime` — target DMA sample time.
- **Side effects:** Advances `s_paintedtime`; writes `paintbuffer` and DMA buffer; dispatches all `S_PaintChannelFrom*` calls; calls `S_TransferPaintBuffer`.
- **Calls:** `Com_Memset`, `S_PaintChannelFromADPCM`, `S_PaintChannelFromWavelet`, `S_PaintChannelFromMuLaw`, `S_PaintChannelFrom16`, `S_TransferPaintBuffer`

## Control Flow Notes
`S_PaintChannels` is called from `snd_dma.c` during the audio update tick (typically once per frame). It drives the entire mix-to-DMA pipeline: clear/fill paint buffer → mix one-shot channels → mix loop channels → transfer to hardware. It loops internally until `s_paintedtime` reaches `endtime`, processing up to `PAINTBUFFER_SIZE` (4096) samples per iteration to bound memory usage.

## External Dependencies
- **`snd_local.h`** — all shared types, channel arrays, DMA state, cvars, and scratch buffer globals
- **`s_channels[MAX_CHANNELS]`**, **`loop_channels`**, **`numLoopChannels`** — defined in `snd_dma.c`
- **`s_paintedtime`**, **`s_rawend`**, **`s_rawsamples`**, **`dma`** — defined in `snd_dma.c`
- **`s_volume`**, **`s_testsound`** — cvars registered in `snd_dma.c`
- **`sfxScratchBuffer`**, **`sfxScratchPointer`**, **`sfxScratchIndex`** — defined in `snd_mem.c`
- **`mulawToShort[256]`** — lookup table defined in `snd_adpcm.c`
- **`S_AdpcmGetSamples`**, **`decodeWavelet`** — defined in `snd_adpcm.c` / `snd_wavelet.c`
- **`S_WriteLinearBlastStereo16`** (Linux x86) — implemented in `unix/snd_mixa.s`
- **`Com_Memset`** — defined in `qcommon`
