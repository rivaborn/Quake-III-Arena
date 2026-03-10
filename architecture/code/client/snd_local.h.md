# code/client/snd_local.h

## File Purpose
Private internal header for Quake III Arena's software sound mixing system. It defines all core data structures, buffer layouts, global state declarations, and internal function prototypes used across the sound subsystem's mixing, spatialization, ADPCM compression, and wavelet/mu-law encoding modules.

## Core Responsibilities
- Define sample buffer structures (`sndBuffer`, `portable_samplepair_t`) for the mixing pipeline
- Define the `sfx_t` sound effect asset type with optional compression metadata
- Define `channel_t` for active playback channels with spatialization state
- Define `dma_t` describing the platform DMA output buffer
- Declare all cross-module globals (channels, listener orientation, cvars, raw sample buffer)
- Declare internal API for sound loading, mixing, spatialization, ADPCM, and wavelet codec functions
- Declare platform-abstraction stubs (`SNDDMA_*`) that must be implemented per OS

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `portable_samplepair_t` | struct | Stereo 32-bit integer sample pair used during mixing before clamping/output |
| `adpcm_state_t` | struct | Tracks ADPCM decoder state: previous sample value and step-size table index |
| `sndBuffer` | struct | Linked-list node holding one chunk of 1024 PCM shorts (or ADPCM state) |
| `sfx_t` | struct | Loaded sound effect: linked list of `sndBuffer` chunks, compression flags, name, LRU timestamp |
| `dma_t` | struct | Describes the DMA ring buffer: channel count, sample count, bit depth, sample rate, raw byte pointer |
| `loopSound_t` | struct | Per-entity looping sound state including origin, velocity, Doppler scale, and active flags |
| `channel_t` | struct | Active mixing channel: start sample, entity/channel IDs, spatialized volumes, Doppler, origin, SFX pointer |
| `wavinfo_t` | struct | WAV file header parse result: format, rate, bit width, channel count, data offset |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_channels` | `channel_t[96]` | global | Active one-shot sound channels |
| `loop_channels` | `channel_t[96]` | global | Active looping sound channels |
| `numLoopChannels` | `int` | global | Count of currently active loop channels |
| `s_paintedtime` | `int` | global | Mixing cursor in samples (absolute) |
| `s_rawend` | `int` | global | End of raw (cinematic/voice) sample data in the ring buffer |
| `listener_forward/right/up` | `vec3_t` | global | Listener orientation vectors for spatialization |
| `dma` | `dma_t` | global | DMA output buffer descriptor |
| `s_rawsamples` | `portable_samplepair_t[16384]` | global | Ring buffer for raw (streaming) audio samples |
| `s_volume/s_nosound/s_khz/s_show/s_mixahead/s_testsound/s_separation` | `cvar_t*` | global | Sound configuration cvars |
| `sfxScratchBuffer` | `short*` | global | Scratch decode buffer shared across sfx decode operations |
| `sfxScratchPointer` | `sfx_t*` | global | Last sfx decoded into scratch (cache key) |
| `sfxScratchIndex` | `int` | global | Current read position in scratch buffer |
| `mulawToShort` | `short[256]` | global | Mu-law to 16-bit PCM lookup table |

## Key Functions / Methods

### SNDDMA_Init
- Signature: `qboolean SNDDMA_Init(void)`
- Purpose: Platform-specific DMA initialization; fills `dma` struct
- Inputs: None
- Outputs/Return: `qtrue` on success
- Side effects: Allocates/maps DMA buffer, sets `dma` global
- Calls: Defined in platform layer (win32/linux)
- Notes: Must be called before any mixing

### SNDDMA_GetDMAPos
- Signature: `int SNDDMA_GetDMAPos(void)`
- Purpose: Returns current hardware DMA write cursor in samples
- Inputs: None
- Outputs/Return: Sample offset into `dma.buffer`
- Side effects: None
- Calls: Platform-specific

### S_LoadSound
- Signature: `qboolean S_LoadSound(sfx_t *sfx)`
- Purpose: Loads and optionally compresses a sound asset into `sfx->soundData`
- Inputs: Pointer to an allocated `sfx_t` with `soundName` set
- Outputs/Return: `qtrue` on success; sets `defaultSound` on failure
- Side effects: Allocates `sndBuffer` chain via `SND_malloc`
- Calls: `wavinfo_t` parsing, `S_AdpcmEncodeSound`, `encodeWavelet`, `encodeMuLaw`

### S_PaintChannels
- Signature: `void S_PaintChannels(int endtime)`
- Purpose: Core mixing loop; mixes all active channels into the DMA paint buffer up to `endtime`
- Inputs: Target sample time
- Outputs/Return: void
- Side effects: Writes to `dma.buffer`; advances `s_paintedtime`
- Calls: `S_AdpcmGetSamples`, `decodeWavelet`, `S_Spatialize`

### S_Spatialize
- Signature: `void S_Spatialize(channel_t *ch)`
- Purpose: Computes left/right volume for a channel based on listener orientation and source position
- Inputs: `channel_t*` with origin/entity set
- Outputs/Return: void; writes `ch->leftvol`, `ch->rightvol`
- Side effects: Reads `listener_forward/right/up`, entity origins
- Notes: Doppler scaling applied if `ch->doppler` is set

### S_AdpcmEncodeSound / S_AdpcmGetSamples
- Signature: `void S_AdpcmEncodeSound(sfx_t*, short*)` / `void S_AdpcmGetSamples(sndBuffer*, short*)`
- Purpose: ADPCM compress on load; decode on playback from `sndBuffer` chain
- Notes: `adpcm_state_t` embedded in each `sndBuffer` node carries delta state

### SND_malloc / SND_free / SND_setup
- Notes: Internal sndBuffer pool allocator. `SND_setup` initializes the pool; `SND_malloc`/`SND_free` allocate/release `sndBuffer` nodes. Prevents per-frame heap allocation during mixing.

### S_FreeOldestSound
- Signature: `void S_FreeOldestSound(void)`
- Purpose: Evicts the least-recently-used `sfx_t` to reclaim `sndBuffer` memory
- Side effects: Frees `sndBuffer` chain, resets `sfx->inMemory`

## Control Flow Notes
- **Init**: `SNDDMA_Init` → `SND_setup` (pool) → cvars registered
- **Per-frame**: `S_Update` (snd_dma.c) calls `S_Spatialize` per channel, then `S_PaintChannels` to mix into the DMA ring buffer, then `SNDDMA_Submit`
- **Asset load**: `S_RegisterSound` → `S_LoadSound` → optional encode (ADPCM/wavelet/mu-law)
- **Shutdown**: `SNDDMA_Shutdown` releases DMA resources

## External Dependencies
- `q_shared.h` — `vec3_t`, `qboolean`, `cvar_t`, `byte`, `MAX_QPATH`
- `qcommon.h` — `Z_Malloc`/`S_Malloc`, `Cvar_Get`, `FS_ReadFile`, `Com_Printf`
- `snd_public.h` — public sound API declarations consumed by client layer
- `SNDDMA_*` functions — defined elsewhere in platform-specific files (`win_snd.c`, `linux_snd.c`, `snd_null.c`)
- `mulawToShort[]` — defined in `snd_adpcm.c` or `snd_wavelet.c`
