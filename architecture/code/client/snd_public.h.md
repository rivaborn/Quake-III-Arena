# code/client/snd_public.h

## File Purpose
Public interface header for the Quake III Arena sound system, exposing all externally callable sound functions to other engine subsystems (client, cgame, etc.). It declares the full lifecycle API for sound playback, looping sounds, spatialization, and background music.

## Core Responsibilities
- Declare sound system initialization and shutdown entry points
- Expose one-shot and looping 3D spatialized sound playback functions
- Provide background music track control (intro + loop)
- Declare raw PCM sample injection for cinematics and VoIP
- Define entity-based position update and reverberation/spatialization calls
- Expose sound registration (asset loading) interface
- Provide utility/diagnostic functions (free memory display, buffer clearing)

## Key Types / Data Structures
None defined in this file. Uses types defined elsewhere (`sfxHandle_t`, `vec3_t`, `qboolean`, `byte`).

## Global / File-Static State
None.

## Key Functions / Methods

### S_Init
- Signature: `void S_Init(void)`
- Purpose: Initializes the sound system (device, buffers, mixer).
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates sound hardware/software resources, sets up DMA or equivalent.
- Calls: Not inferable from this file.
- Notes: Must be called before any other S_* function.

### S_Shutdown
- Signature: `void S_Shutdown(void)`
- Purpose: Tears down the sound system and frees resources.
- Inputs: None
- Outputs/Return: None
- Side effects: Releases hardware/software audio resources.
- Calls: Not inferable from this file.

### S_StartSound
- Signature: `void S_StartSound(vec3_t origin, int entnum, int entchannel, sfxHandle_t sfx)`
- Purpose: Plays a one-shot sound, either at a fixed world origin or dynamically tracked to an entity.
- Inputs: `origin` — world position (NULL = use entity position dynamically); `entnum` — entity index; `entchannel` — channel slot; `sfx` — registered sound handle.
- Outputs/Return: None
- Side effects: Allocates a sound channel, begins mixing.
- Notes: If `origin` is NULL, position is derived each frame from the entity.

### S_StartLocalSound
- Signature: `void S_StartLocalSound(sfxHandle_t sfx, int channelNum)`
- Purpose: Plays a non-spatialized (2D/UI) sound at full volume.
- Inputs: `sfx` — sound handle; `channelNum` — channel slot.
- Outputs/Return: None
- Side effects: Allocates a local (non-positional) channel.

### S_StartBackgroundTrack / S_StopBackgroundTrack
- Signature: `void S_StartBackgroundTrack(const char *intro, const char *loop)` / `void S_StopBackgroundTrack(void)`
- Purpose: Starts streaming a music track (plays `intro` once, then loops `loop`); stops it.
- Inputs: `intro`, `loop` — file paths to audio tracks.
- Side effects: Opens streaming audio file handles, allocates streaming buffer.

### S_RawSamples
- Signature: `void S_RawSamples(int samples, int rate, int width, int channels, const byte *data, float volume)`
- Purpose: Injects raw PCM audio (used by cinematics and network voice).
- Inputs: `samples` count, `rate` (Hz), `width` (bits/8), `channels` (mono/stereo), raw `data` pointer, `volume` scalar.
- Side effects: Writes directly into the mix buffer bypassing the normal sound pipeline.
- Notes: `volume` 1.0 = unity gain.

### S_Respatialize
- Signature: `void S_Respatialize(int entityNum, const vec3_t origin, vec3_t axis[3], int inwater)`
- Purpose: Recomputes relative volumes for all active sounds based on the listener's entity, position, orientation, and medium.
- Inputs: Listener entity, position, 3×3 orientation axes, underwater flag.
- Side effects: Modifies per-channel volume/pan state for the current frame.
- Notes: Called once per frame by the client.

### S_RegisterSound
- Signature: `sfxHandle_t S_RegisterSound(const char *sample, qboolean compressed)`
- Purpose: Loads and registers a sound asset, returning a handle.
- Inputs: `sample` — asset path; `compressed` — whether to use compressed (ADPCM/wavelet) storage.
- Outputs/Return: Always returns a valid `sfxHandle_t`; creates a placeholder if the file is missing.
- Notes: Never returns an invalid handle — safe to call for optional sounds.

### S_Update
- Signature: `void S_Update(void)`
- Purpose: Per-frame sound mixer tick — updates positions, mixes channels, submits to DMA.
- Side effects: Drives the entire audio pipeline each frame.

### Notes
- `S_ClearLoopingSounds`, `S_AddLoopingSound`, `S_AddRealLoopingSound`, `S_StopLoopingSound` — looping sound management; all looping sounds must be re-added each frame before `S_Update`.
- `S_UpdateEntityPosition` — informs the sound system of an entity's new world position.
- `S_StopAllSounds`, `S_DisableSounds`, `S_ClearSoundBuffer` — bulk stop/reset utilities.
- `S_BeginRegistration` — marks the start of an asset registration pass.
- `SNDDMA_Activate` — platform-level DMA reactivation (e.g., on window focus restore).
- `S_UpdateBackgroundTrack` — per-frame streaming update for the music track.

## Control Flow Notes
This header sits at the boundary between the **client layer** and the **sound subsystem**. `S_Init` is called during client startup; `S_Update` and `S_Respatialize` are called every client frame; `S_Shutdown` is called on exit. Looping sounds follow a clear per-frame protocol: `S_ClearLoopingSounds` → add all loops → `S_Update`.

## External Dependencies
- `vec3_t`, `qboolean`, `byte` — defined in `q_shared.h`
- `sfxHandle_t` — defined in `q_shared.h` or `snd_local.h`
- All function bodies defined in `snd_dma.c`, `snd_mix.c`, `snd_mem.c` (and platform DMA backends)
