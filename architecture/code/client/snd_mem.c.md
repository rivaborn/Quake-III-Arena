# code/client/snd_mem.c

## File Purpose
Implements the sound memory manager and WAV file loader for Quake III Arena's audio system. It manages a fixed-size pool of `sndBuffer` chunks via a free-list allocator, parses WAV headers, and resamples raw PCM audio to match the engine's DMA output rate.

## Core Responsibilities
- Initialize and manage a slab-based free-list allocator for `sndBuffer` chunks
- Parse RIFF/WAV file headers to extract format metadata (`wavinfo_t`)
- Resample PCM audio data (8-bit or 16-bit, mono) from source rate to `dma.speed`
- Load and decode sound assets into `sfx_t` structures, optionally applying ADPCM compression
- Report free/used sound memory statistics

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `sndBuffer` | struct (defined in `snd_local.h`) | Fixed-size chunk of 1024 PCM samples + linked-list pointer + ADPCM state |
| `sfx_t` | struct (defined in `snd_local.h`) | Sound effect descriptor; holds linked list of `sndBuffer` chunks and metadata |
| `wavinfo_t` | struct (defined in `snd_local.h`) | Parsed WAV header fields: rate, width, channels, sample count, data offset |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `buffer` | `sndBuffer *` | static (file) | Base pointer to the entire allocated slab |
| `freelist` | `sndBuffer *` | static (file) | Head of the free-list of available `sndBuffer` chunks |
| `inUse` | `int` | static (file) | Bytes currently free (decremented on alloc, incremented on free) |
| `totalInUse` | `int` | static (file) | Cumulative bytes ever allocated |
| `sfxScratchBuffer` | `short *` | global | Temporary scratch buffer for raw resampled samples during load |
| `sfxScratchPointer` | `sfx_t *` | global | Tracks which sfx currently owns the scratch buffer |
| `sfxScratchIndex` | `int` | global | Current read index into the scratch buffer |
| `data_p` | `byte *` | static (file) | Cursor into the WAV byte stream during parsing |
| `iff_end` | `byte *` | static (file) | End-of-file boundary for WAV parsing |
| `last_chunk` | `byte *` | static (file) | Start of last found chunk, for sequential scanning |
| `iff_data` | `byte *` | static (file) | Start of current IFF section |
| `iff_chunk_len` | `int` | static (file) | Length of the most recently found IFF chunk |

## Key Functions / Methods

### SND_setup
- **Signature:** `void SND_setup(void)`
- **Purpose:** Allocates the sound memory pool and scratch buffer; initializes the free-list.
- **Inputs:** None (reads `com_soundMegs` cvar).
- **Outputs/Return:** None.
- **Side effects:** Calls `malloc` twice; sets `buffer`, `freelist`, `sfxScratchBuffer`, `inUse`.
- **Calls:** `Cvar_Get`, `malloc`, `Com_Printf`
- **Notes:** Pool size = `com_soundMegs * 1536` buffers. Free-list is built in reverse order through the slab.

### SND_malloc
- **Signature:** `sndBuffer* SND_malloc(void)`
- **Purpose:** Pops one `sndBuffer` from the free-list; evicts the oldest sound if the list is empty.
- **Inputs:** None.
- **Outputs/Return:** Pointer to an available `sndBuffer`.
- **Side effects:** Decrements `inUse`, increments `totalInUse`; may call `S_FreeOldestSound` (defined in `snd_dma.c`).
- **Calls:** `S_FreeOldestSound`
- **Notes:** Uses `goto redo` to retry after eviction; `next` field of returned buffer is cleared to NULL.

### SND_free
- **Signature:** `void SND_free(sndBuffer *v)`
- **Purpose:** Returns a `sndBuffer` to the free-list.
- **Inputs:** `v` — buffer to release.
- **Outputs/Return:** None.
- **Side effects:** Increments `inUse`; prepends `v` to `freelist`.

### GetWavinfo
- **Signature:** `static wavinfo_t GetWavinfo(char *name, byte *wav, int wavlength)`
- **Purpose:** Parses a RIFF/WAV byte array and returns format metadata.
- **Inputs:** `name` — filename for error messages; `wav` — raw file bytes; `wavlength` — byte count.
- **Outputs/Return:** Populated `wavinfo_t`; zeroed on failure.
- **Side effects:** Modifies file-static cursor variables (`data_p`, `iff_data`, etc.).
- **Calls:** `FindChunk`, `FindNextChunk`, `GetLittleShort`, `GetLittleLong`, `Com_Memset`, `Com_Printf`
- **Notes:** Only accepts Microsoft PCM format (format == 1); mono-only enforcement is upstream in `S_LoadSound`.

### ResampleSfx
- **Signature:** `static void ResampleSfx(sfx_t *sfx, int inrate, int inwidth, byte *data, qboolean compressed)`
- **Purpose:** Resamples raw PCM into the engine's output rate, allocating `sndBuffer` chunks linked onto `sfx->soundData`.
- **Inputs:** `sfx` — target; `inrate/inwidth` — source format; `data` — raw PCM bytes.
- **Outputs/Return:** None; mutates `sfx->soundData` and `sfx->soundLength`.
- **Side effects:** Calls `SND_malloc` per chunk; writes `sfx->soundData` linked list.
- **Calls:** `SND_malloc`, `LittleShort`

### ResampleSfxRaw
- **Signature:** `static int ResampleSfxRaw(short *sfx, int inrate, int inwidth, int samples, byte *data)`
- **Purpose:** Resamples raw PCM into a caller-supplied `short` array (used before ADPCM encoding).
- **Inputs:** `sfx` — output buffer; source format parameters; `data` — raw PCM.
- **Outputs/Return:** Number of output samples.
- **Side effects:** None beyond writing `sfx[]`.
- **Calls:** `LittleShort`

### S_LoadSound
- **Signature:** `qboolean S_LoadSound(sfx_t *sfx)`
- **Purpose:** Top-level entry point: reads a WAV file, validates it, resamples it, and optionally ADPCM-encodes it into the sfx chain.
- **Inputs:** `sfx` — descriptor with `soundName` and compression flags set.
- **Outputs/Return:** `qtrue` on success, `qfalse` on failure.
- **Side effects:** Allocates temp hunk memory; calls `FS_ReadFile`/`FS_FreeFile`; populates `sfx->soundData`, `sfx->soundLength`, `sfx->soundCompressionMethod`, `sfx->lastTimeUsed`.
- **Calls:** `FS_ReadFile`, `GetWavinfo`, `Hunk_AllocateTempMemory`, `Com_Milliseconds`, `ResampleSfxRaw`, `S_AdpcmEncodeSound`, `ResampleSfx`, `Hunk_FreeTempMemory`, `FS_FreeFile`, `Com_Printf`, `Com_DPrintf`
- **Notes:** Stereo WAVs are rejected. 8-bit and non-22kHz files produce warnings but still load. Mu-law and wavelet paths are compiled out (`#if 0`).

## Control Flow Notes
`SND_setup` is called once at sound system initialization (from `S_Init` in `snd_dma.c`). `S_LoadSound` is called on-demand when a sound is first referenced; it is not part of the per-frame update path. `SND_malloc`/`SND_free` are called throughout the mixing and loading code whenever sample chunks are needed or released.

## External Dependencies
- **Includes:** `snd_local.h` → `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state; `dma.speed` used for resampling ratio
  - `S_FreeOldestSound` — eviction policy, defined in `snd_dma.c`
  - `S_AdpcmEncodeSound` — ADPCM encoder, defined in `snd_adpcm.c`
  - `LittleShort` — endian swap macro, from `q_shared.h`
  - `FS_ReadFile`, `FS_FreeFile`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Com_Milliseconds`, `Cvar_Get`, `Com_Printf`, `Com_DPrintf`, `Com_Memset` — engine common layer
