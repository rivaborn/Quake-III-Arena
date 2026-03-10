# code/client/snd_dma.c

## File Purpose
Main control module for the Quake III Arena software-mixed sound system. It manages sound channel allocation, spatialization, looping sounds, background music streaming, and drives the DMA mixing pipeline each frame.

## Core Responsibilities
- Initialize and shut down the sound system via `SNDDMA_*` platform layer
- Register, cache, and evict sound assets (`sfx_t`) from memory
- Allocate and manage `channel_t` slots for one-shot and looping sounds
- Spatialize 3D sound channels using listener position and orientation
- Stream background music from WAV files into the raw sample buffer
- Drive the mixing pipeline (`S_PaintChannels`) each frame via `S_Update_`
- Handle Doppler scaling for looping sounds tied to moving entities

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `channel_t` | struct | Active sound playback slot with volume, entity binding, sfx pointer |
| `sfx_t` | struct | Sound asset descriptor with linked-list of `sndBuffer` chunks |
| `dma_t` | struct | DMA buffer descriptor: channels, speed, samplebits, submission chunk |
| `loopSound_t` | struct | Per-entity looping sound state including Doppler parameters |
| `portable_samplepair_t` | struct | Stereo 32-bit sample pair for the raw streaming buffer |
| `wavinfo_t` | struct | Parsed WAV header info used for background track |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_channels[MAX_CHANNELS]` | `channel_t[96]` | global | Active one-shot sound channels |
| `loop_channels[MAX_CHANNELS]` | `channel_t[96]` | global | Spatialized looping sound channels built each frame |
| `numLoopChannels` | `int` | global | Count of active loop channels |
| `dma` | `dma_t` | global | DMA hardware descriptor shared with mixer |
| `s_soundtime` | `int` | global | Current DMA playback position in sample pairs |
| `s_paintedtime` | `int` | global | How far ahead mixing has been painted |
| `s_rawend` | `int` | global | End of valid data in `s_rawsamples` ring buffer |
| `s_rawsamples[MAX_RAW_SAMPLES]` | `portable_samplepair_t[]` | global | Ring buffer for streamed audio (music, cinematics) |
| `s_knownSfx[MAX_SFX]` | `sfx_t[4096]` | global | Flat array of all registered sound assets |
| `sfxHash[LOOP_HASH]` | `sfx_t*[128]` | static | Hash table for fast sfx lookup by name |
| `loopSounds[MAX_GENTITIES]` | `loopSound_t[]` | static | Per-entity loop sound state |
| `freelist` | `channel_t*` | static | Intrusive free-list head for channel pool |
| `s_backgroundFile` | `fileHandle_t` | static | Handle for currently streaming background WAV |
| `s_backgroundInfo` | `wavinfo_t` | static | Parsed header of background WAV |
| `s_backgroundSamples` | `int` | static | Remaining samples in background file |
| `s_backgroundLoop[MAX_QPATH]` | `char[]` | static | Loop filename for background track |
| `s_soundStarted` | `int` | static | Non-zero when sound system is running |
| `s_soundMuted` | `qboolean` | static | Suppresses all output when true |
| `listener_number` | `int` | static | Entity index of the listener |
| `listener_origin` | `vec3_t` | static | World position of listener |
| `listener_axis[3]` | `vec3_t[3]` | static | Orientation basis of listener |

## Key Functions / Methods

### S_Init
- Signature: `void S_Init(void)`
- Purpose: Initialize the sound subsystem: register cvars, console commands, call `SNDDMA_Init`, and clear state.
- Inputs: None (reads cvars)
- Outputs/Return: void
- Side effects: Sets `s_soundStarted`, clears `sfxHash`, calls `S_StopAllSounds`
- Calls: `Cvar_Get`, `Cmd_AddCommand`, `SNDDMA_Init`, `S_StopAllSounds`, `S_SoundInfo_f`

### S_Shutdown
- Signature: `void S_Shutdown(void)`
- Purpose: Tear down the sound system and unregister console commands.
- Side effects: Calls `SNDDMA_Shutdown`, clears `s_soundStarted`

### S_RegisterSound
- Signature: `sfxHandle_t S_RegisterSound(const char *name, qboolean compressed)`
- Purpose: Find or create an `sfx_t` entry and load its audio data into memory.
- Inputs: Sound path `name`; `compressed` is forced to `qfalse`
- Outputs/Return: Index into `s_knownSfx`, or 0 on failure
- Side effects: May allocate `sndBuffer` chains via `S_memoryLoad`
- Calls: `S_FindName`, `S_memoryLoad`

### S_StartSound
- Signature: `void S_StartSound(vec3_t origin, int entityNum, int entchannel, sfxHandle_t sfxHandle)`
- Purpose: Queue a one-shot sound on an available channel, with deduplication and channel-stealing logic.
- Inputs: World origin (or NULL for entity-tracked), entity number, channel slot, sfx handle
- Outputs/Return: void
- Side effects: Allocates from `freelist` or steals oldest channel; writes to `s_channels`
- Notes: Deduplicates sounds within 50 ms; protects `CHAN_ANNOUNCER` and `listener_number` channels from stealing.

### S_SpatializeOrigin
- Signature: `void S_SpatializeOrigin(vec3_t origin, int master_vol, int *left_vol, int *right_vol)`
- Purpose: Compute left/right volumes from a world-space origin relative to the listener.
- Inputs: Source position, master volume (0–127)
- Outputs/Return: Writes `left_vol` and `right_vol` (0–255)
- Side effects: None
- Notes: Uses `SOUND_FULLVOLUME=80` units before attenuation begins; mono DMA bypasses panning.

### S_Respatialize
- Signature: `void S_Respatialize(int entityNum, const vec3_t head, vec3_t axis[3], int inwater)`
- Purpose: Update listener position/orientation, then respatialize all active channels and rebuild loop channels.
- Side effects: Updates `listener_*` globals; calls `S_SpatializeOrigin` per channel; calls `S_AddLoopSounds`

### S_Update
- Signature: `void S_Update(void)`
- Purpose: Per-frame entry point: feed background music into raw buffer, then drive the mix pipeline.
- Calls: `S_UpdateBackgroundTrack`, `S_Update_`

### S_Update_
- Signature: `void S_Update_(void)`
- Purpose: Compute mix-ahead window, call `S_PaintChannels` to fill DMA buffer, submit via `SNDDMA_Submit`.
- Side effects: Calls `S_GetSoundtime`, `S_ScanChannelStarts`, `S_PaintChannels`, `SNDDMA_BeginPainting`, `SNDDMA_Submit`
- Notes: Clamps mix rate to ~85 Hz minimum; aligns `endtime` to `submission_chunk` boundary.

### S_RawSamples
- Signature: `void S_RawSamples(int samples, int rate, int width, int s_channels, const byte *data, float volume)`
- Purpose: Resample and volume-scale incoming PCM data (music, cinematics) into the `s_rawsamples` ring buffer.
- Side effects: Advances `s_rawend`; may overflow-warn

### S_UpdateBackgroundTrack
- Signature: `void S_UpdateBackgroundTrack(void)`
- Purpose: Stream chunks from the open WAV background file into `s_rawsamples` each frame; loop when exhausted.
- Side effects: Reads from `s_backgroundFile` via `Sys_StreamedRead`; calls `S_RawSamples`; restarts track on loop

### S_AddLoopSounds
- Signature: `void S_AddLoopSounds(void)`
- Purpose: Spatialize all active `loopSounds` entries, merging duplicates, into `loop_channels` for the current frame.
- Side effects: Writes `loop_channels[]` and `numLoopChannels`

### S_FreeOldestSound
- Signature: `void S_FreeOldestSound(void)`
- Purpose: Evict the least-recently-used in-memory sfx to reclaim `sndBuffer` allocations.
- Side effects: Calls `SND_free` on each buffer chunk; marks sfx as not in memory

## Control Flow Notes
- **Init**: `S_Init` → `SNDDMA_Init` → `S_StopAllSounds`
- **Per-frame**: `S_Respatialize` (called by cgame to update listener) → `S_Update` (called by client frame) → `S_Update_` → `S_PaintChannels` (defined in `snd_mix.c`) → `SNDDMA_Submit`
- **Sound start**: game/cgame calls `S_StartSound` / `S_AddLoopingSound`; volumes are resolved at `S_Respatialize` time
- **Shutdown**: `S_Shutdown` → `SNDDMA_Shutdown`

## External Dependencies
- `snd_local.h`: `sfx_t`, `channel_t`, `dma_t`, `loopSound_t`, `SNDDMA_*`, `S_PaintChannels`, `SND_malloc/free`, `S_LoadSound`
- `client.h`: `cls.framecount` (Doppler frame tracking)
- **Defined elsewhere**: `SNDDMA_Init/Shutdown/GetDMAPos/BeginPainting/Submit` (platform layer: `win_snd.c` / `linux_snd.c`), `S_PaintChannels` (`snd_mix.c`), `S_LoadSound` (`snd_mem.c`), `Sys_BeginStreamedFile/StreamedRead/EndStreamedFile` (OS layer), `VectorRotate`, `DistanceSquared` (math), `Com_Milliseconds`, `FS_Read/FOpenFileRead/FCloseFile`
