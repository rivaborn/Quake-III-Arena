# code/client/snd_adpcm.c

## File Purpose
Implements Intel/DVI ADPCM (Adaptive Differential Pulse-Code Modulation) audio compression and decompression for Quake III Arena's sound system. It encodes raw PCM audio into a 4-bit-per-sample ADPCM format and decodes it back, and provides the glue functions to store/retrieve ADPCM-compressed sound data in the engine's chunked `sndBuffer` system.

## Core Responsibilities
- Encode 16-bit PCM samples into 4-bit ADPCM nibbles (`S_AdpcmEncode`)
- Decode 4-bit ADPCM nibbles back to 16-bit PCM samples (`S_AdpcmDecode`)
- Calculate memory requirements for ADPCM-compressed sound assets (`S_AdpcmMemoryNeeded`)
- Retrieve decoded samples from a single `sndBuffer` chunk (`S_AdpcmGetSamples`)
- Encode an entire `sfx_t` sound asset into a linked list of `sndBuffer` chunks (`S_AdpcmEncodeSound`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `adpcm_state_t` | typedef struct (`adpcm_state`) | Carries codec state between calls: last predicted sample and step-table index |
| `sndBuffer` | typedef struct (`sndBuffer_s`) | Fixed-size chunk of sound data with embedded `adpcm_state_t` header and a `next` pointer for chaining |
| `sfx_t` | typedef struct (`sfx_s`) | Top-level sound effect descriptor; owns the linked list of `sndBuffer` chunks |
| `wavinfo_t` | typedef struct | Describes source WAV metadata (rate, samples, channels) used for memory estimation |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `indexTable[16]` | `static int[16]` | file-static | Maps 4-bit ADPCM delta values to step-index adjustments |
| `stepsizeTable[89]` | `static int[89]` | file-static | 89-entry quantization step-size table per IMA ADPCM spec |

## Key Functions / Methods

### S_AdpcmEncode
- **Signature:** `void S_AdpcmEncode( short indata[], char outdata[], int len, struct adpcm_state *state )`
- **Purpose:** Encodes `len` 16-bit PCM samples into packed 4-bit ADPCM nibbles (two per output byte).
- **Inputs:** `indata` — source PCM buffer; `outdata` — destination byte buffer (must be ≥ `len/2` bytes); `len` — sample count; `state` — encoder state (updated in place).
- **Outputs/Return:** void; writes packed nibbles into `outdata`; updates `state->sample` and `state->index`.
- **Side effects:** Mutates `*state`.
- **Calls:** None (pure arithmetic, table lookups).
- **Notes:** Odd `len` flushes the final nibble as the high nibble of a partial byte. Uses bit-shift approximation rather than division for speed.

### S_AdpcmDecode
- **Signature:** `void S_AdpcmDecode( const char indata[], short *outdata, int len, struct adpcm_state *state )`
- **Purpose:** Decodes `len` ADPCM nibbles back into 16-bit PCM samples.
- **Inputs:** `indata` — packed nibble source; `outdata` — destination `short` array (must be ≥ `len`); `len` — number of samples to decode; `state` — decoder state.
- **Outputs/Return:** void; writes decoded samples to `outdata`; updates `state`.
- **Side effects:** Mutates `*state`.
- **Calls:** None.
- **Notes:** Marked `/* static */` in source — originally internal, promoted to external linkage for `S_AdpcmGetSamples`.

### S_AdpcmMemoryNeeded
- **Signature:** `int S_AdpcmMemoryNeeded( const wavinfo_t *info )`
- **Purpose:** Calculates total bytes needed to store a sound in ADPCM format, including per-block `adpcm_state_t` headers.
- **Inputs:** `info` — WAV metadata (rate, sample count).
- **Outputs/Return:** Total byte count (sample data + block headers).
- **Side effects:** Reads global `dma.speed` for rate scaling.
- **Calls:** None.
- **Notes:** Rate conversion via float division; blocks sized to `PAINTBUFFER_SIZE` (4096 samples).

### S_AdpcmGetSamples
- **Signature:** `void S_AdpcmGetSamples(sndBuffer *chunk, short *to)`
- **Purpose:** Decodes one `sndBuffer` chunk's worth of ADPCM data into a `short` PCM output buffer.
- **Inputs:** `chunk` — a single buffer node (provides header state + raw ADPCM bytes); `to` — destination PCM buffer.
- **Outputs/Return:** void; writes `SND_CHUNK_SIZE_BYTE*2` decoded samples into `to`.
- **Side effects:** None on global state; `state` is stack-local.
- **Calls:** `S_AdpcmDecode`.

### S_AdpcmEncodeSound
- **Signature:** `void S_AdpcmEncodeSound( sfx_t *sfx, short *samples )`
- **Purpose:** Encodes an entire PCM sample array into a linked list of `sndBuffer` chunks attached to `sfx->soundData`.
- **Inputs:** `sfx` — sound asset to populate; `samples` — raw 16-bit PCM input.
- **Outputs/Return:** void; populates `sfx->soundData` linked list.
- **Side effects:** Calls `SND_malloc()` (allocates `sndBuffer` nodes); mutates `sfx->soundData` chain.
- **Calls:** `SND_malloc`, `S_AdpcmEncode`.
- **Notes:** Each chunk stores its initial `adpcm_state_t` as a header so decoding can start from any block independently. Chunk payload is `SND_CHUNK_SIZE_BYTE*2` samples (2048 samples per chunk).

## Control Flow Notes
This file is used only during **load time** (`S_AdpcmEncodeSound` — called when a WAV is loaded and `soundCompressed` is set) and during **mix time** (`S_AdpcmGetSamples` — called by the paint/mix path in `snd_mix.c` to stream decoded PCM into the mix buffer). It has no per-frame init or shutdown involvement.

## External Dependencies
- **Includes:** `snd_local.h` → pulls in `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state providing `dma.speed`
  - `SND_malloc()` — sndBuffer allocator (defined in `snd_mem.c`)
  - `PAINTBUFFER_SIZE`, `SND_CHUNK_SIZE_BYTE` — macros from `snd_local.h`
  - `adpcm_state_t`, `sndBuffer`, `sfx_t`, `wavinfo_t` — types from `snd_local.h`
