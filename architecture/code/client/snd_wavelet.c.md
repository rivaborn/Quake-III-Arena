# code/client/snd_wavelet.c

## File Purpose
Implements wavelet-based and mu-law audio compression/decompression for Quake III's sound system. It encodes PCM audio data into compact `sndBuffer` chunks using either a Daubechies-4 wavelet transform followed by mu-law quantization, or mu-law encoding alone with dithered error feedback.

## Core Responsibilities
- Apply forward/inverse Daubechies-4 (daub4) wavelet transform to float sample arrays
- Drive multi-resolution wavelet decomposition/reconstruction via `wt1`
- Encode 16-bit PCM samples to 8-bit mu-law bytes (`MuLawEncode`)
- Decode 8-bit mu-law bytes back to 16-bit PCM (`MuLawDecode`)
- Build and cache the `mulawToShort[256]` lookup table on first use
- Compress an `sfx_t` sound asset into linked `sndBuffer` chunks (`encodeWavelet`, `encodeMuLaw`)
- Decompress `sndBuffer` chunks back to PCM for mixing (`decodeWavelet`, `decodeMuLaw`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `sfx_t` | struct (defined in `snd_local.h`) | Sound asset; holds linked list of `sndBuffer` chunks and metadata |
| `sndBuffer` | struct (defined in `snd_local.h`) | Fixed-size chunk of compressed audio samples plus linked-list pointer |
| `NXStream` | typedef (`byte`) | Byte-stream alias used for `NXPutc` output buffer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `mulawToShort` | `short[256]` | global (extern) | Precomputed mu-law byte → 16-bit PCM lookup table |
| `madeTable` | `static qboolean` | static | Guards one-time initialization of `mulawToShort` |
| `NXStreamCount` | `static int` | static (file) | Write cursor for the `NXPutc` byte stream; declared `static int` but not marked static in code — effectively file-scoped via translation unit |

## Key Functions / Methods

### daub4
- **Signature:** `void daub4(float b[], unsigned long n, int isign)`
- **Purpose:** Single-level Daubechies-4 wavelet filter pass over `n` samples in `b`.
- **Inputs:** `b` — 0-based float array; `n` — sample count (must be ≥ 4, power of 2); `isign ≥ 0` = forward, `< 0` = inverse.
- **Outputs/Return:** Writes result back into `b[]` in-place.
- **Side effects:** Stack-allocates `wksp[4097]`; no heap allocation.
- **Calls:** None.
- **Notes:** Uses 1-based indexing internally via `a = b - 1` (Numerical Recipes convention). Buffer overflow risk if `n > 4096`.

### wt1
- **Signature:** `void wt1(float a[], unsigned long n, int isign)`
- **Purpose:** Full multi-resolution wavelet transform — iterates `daub4` across all dyadic sub-bands down to `n/4`.
- **Inputs:** `a` — float sample array; `n` — total sample count; `isign` — forward/inverse direction.
- **Outputs/Return:** Modifies `a[]` in-place.
- **Side effects:** None beyond modifying `a`.
- **Calls:** `daub4`
- **Notes:** Forward pass descends (`nn >>= 1`); inverse ascends (`nn <<= 1`). Minimum sub-band length is `n/4`.

### MuLawEncode
- **Signature:** `byte MuLawEncode(short s)`
- **Purpose:** Compresses a 16-bit signed PCM sample to an 8-bit mu-law byte.
- **Inputs:** 16-bit signed PCM sample.
- **Outputs/Return:** 8-bit mu-law encoded byte.
- **Side effects:** None.
- **Calls:** `numBits[]` table lookup.
- **Notes:** Adds bias of 128+4 before encoding; clamps to 32767.

### MuLawDecode
- **Signature:** `short MuLawDecode(byte uLaw)`
- **Purpose:** Expands an 8-bit mu-law byte back to a 16-bit PCM sample.
- **Inputs:** 8-bit mu-law byte.
- **Outputs/Return:** 16-bit signed PCM.
- **Side effects:** None.

### encodeWavelet
- **Signature:** `void encodeWavelet(sfx_t *sfx, short *packets)`
- **Purpose:** Encodes a full PCM sound asset into linked `sndBuffer` chunks using wavelet + mu-law compression.
- **Inputs:** `sfx` — target sound asset (modified); `packets` — source 16-bit PCM buffer.
- **Outputs/Return:** Populates `sfx->soundData` linked list.
- **Side effects:** Calls `SND_malloc()` (heap allocation per chunk); initializes `mulawToShort` table on first call; sets `madeTable`.
- **Calls:** `SND_malloc`, `wt1`, `MuLawEncode`
- **Notes:** Chunk size is `min(samples, SND_CHUNK_SIZE*2)`, minimum 4; float workspace capped at 4097.

### decodeWavelet
- **Signature:** `void decodeWavelet(sndBuffer *chunk, short *to)`
- **Purpose:** Decompresses one `sndBuffer` chunk from mu-law + wavelet back to PCM.
- **Inputs:** `chunk` — compressed buffer; `to` — destination PCM array (may be NULL).
- **Outputs/Return:** Writes `chunk->size` PCM samples into `to`.
- **Side effects:** None (read-only on chunk).
- **Calls:** `wt1`

### encodeMuLaw
- **Signature:** `void encodeMuLaw(sfx_t *sfx, short *packets)`
- **Purpose:** Encodes PCM to mu-law only (no wavelet), with dithered error feedback to reduce quantization noise.
- **Inputs/Outputs:** Same pattern as `encodeWavelet`.
- **Side effects:** `SND_malloc`, sets `madeTable`.
- **Calls:** `SND_malloc`, `MuLawEncode`
- **Notes:** `grade` accumulates quantization error and feeds it forward each sample.

### decodeMuLaw
- **Signature:** `void decodeMuLaw(sndBuffer *chunk, short *to)`
- **Purpose:** Direct mu-law lookup decode into PCM — no wavelet inverse.
- **Calls:** Table lookup only (`mulawToShort`).

## Control Flow Notes
This file is used during **sound asset loading** (encode path called once when a sound is loaded into memory) and during **mixing** (decode path called per-chunk each frame when mixing channels). It is not called per-frame for the encode path. `SND_malloc` ties into the sound memory pool managed in `snd_mem.c`. The `NXPutc`/`NXStreamCount` pair appears vestigial — declared but not called from any encode/decode path in this file.

## External Dependencies
- `snd_local.h` — `sfx_t`, `sndBuffer`, `SND_CHUNK_SIZE`, `SND_malloc`, `NXStream`, `qboolean`, `byte`, `short`
- `SND_malloc` — defined in `snd_mem.c`
- `myftol` — declared but not called in this file; defined elsewhere (platform float-to-long helper)
- `numBits[256]` — file-static lookup table for bit-count of byte values
