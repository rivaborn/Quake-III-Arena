# code/client/cl_cin.c

## File Purpose
Implements RoQ video cinematic playback for Quake III Arena, handling decoding of RoQ-format video frames (VQ-compressed), YUV-to-RGB color conversion, audio decompression (RLL-encoded mono/stereo), and rendering of cinematics to the screen or in-game surfaces.

## Core Responsibilities
- Parse and decode RoQ video file format (header, codebook, VQ frames, audio packets)
- Perform YUV→RGB(16-bit and 32-bit) color space conversion using precomputed lookup tables
- Decode RLL-encoded audio (mono/stereo variants) into PCM samples and feed to the sound system
- Manage up to 16 simultaneous video handles (`cinTable[MAX_VIDEO_HANDLES]`)
- Build and cache the quad-tree blitting structure for VQ frame rendering
- Handle looping, hold-at-end, in-game shader video, and game-state transitions
- Upload decoded frames to the renderer via `re.DrawStretchRaw` / `re.UploadCinematic`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cinematics_t` | struct | Shared per-decode-session state: line buffer, file read buffer, sqr table, motion comp table, quad status pointers |
| `cin_cache` | struct | Per-handle playback state: filename, dimensions, playback flags, VQ function pointers, timing, frame counters |
| `e_status` | typedef (enum, defined elsewhere) | Playback status: `FMV_PLAY`, `FMV_EOF`, `FMV_IDLE`, `FMV_LOOPED` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cin` | `cinematics_t` | static/global | Shared decode workspace (linbuf, sqrTable, qStatus, mcomp) |
| `cinTable[16]` | `cin_cache[]` | static | Per-handle playback state |
| `currentHandle` | `int` | static | Active handle for most internal operations |
| `CL_handle` | `int` | static | Handle for the "system" (full-screen) cinematic |
| `ROQ_YY_tab`…`ROQ_VR_tab` | `long[256]` | static | YUV→RGB conversion lookup tables |
| `vq2`, `vq4`, `vq8` | `unsigned short[]` | static | VQ codebook pixel data at 2×2, 4×4, 8×8 granularity |

## Key Functions / Methods

### CIN_PlayCinematic
- **Signature:** `int CIN_PlayCinematic(const char *arg, int x, int y, int w, int h, int systemBits)`
- **Purpose:** Opens and begins playback of a RoQ video file.
- **Inputs:** Filename, screen position/size, bitflags (`CIN_loop`, `CIN_hold`, `CIN_system`, `CIN_silent`, `CIN_shader`)
- **Outputs/Return:** Handle index (0–15) on success, -1 on failure
- **Side effects:** Opens file via `FS_FOpenFileRead`, starts streamed I/O via `Sys_BeginStreamedFile`, sets `cls.state = CA_CINEMATIC` if system cinematic, closes UI menu, resets `s_rawend`
- **Calls:** `CIN_HandleForVideo`, `initRoQ`, `RoQ_init`, `CIN_SetExtents`, `CIN_SetLooping`, `VM_Call`, `Con_Close`

### CIN_RunCinematic
- **Signature:** `e_status CIN_RunCinematic(int handle)`
- **Purpose:** Per-frame update: advances the cinematic by decoding as many RoQ frames as needed to match wall-clock time.
- **Inputs:** Handle index
- **Outputs/Return:** Current `e_status`
- **Side effects:** Calls `RoQInterrupt` in a loop, may call `RoQReset` or `RoQShutdown` on EOF/loop
- **Calls:** `CL_ScaledMilliseconds`, `RoQInterrupt`, `RoQReset`, `RoQShutdown`

### CIN_StopCinematic
- **Signature:** `e_status CIN_StopCinematic(int handle)`
- **Purpose:** Stops playback of a specific handle; triggers `RoQShutdown`.
- **Side effects:** Sets status to `FMV_EOF`, closes file if in shutdown path

### CIN_DrawCinematic
- **Signature:** `void CIN_DrawCinematic(int handle)`
- **Purpose:** Submits the decoded frame to the renderer, downsampling to 256×256 when the hardware (Rage Pro, Voodoo) requires it.
- **Side effects:** Allocates/frees temp hunk memory for downsample; calls `re.DrawStretchRaw`; clears `dirty` flag

### CIN_UploadCinematic
- **Signature:** `void CIN_UploadCinematic(int handle)`
- **Purpose:** Uploads the current frame as a texture for in-game surface (shader) video.
- **Side effects:** Calls `re.UploadCinematic`; manages `playonwalls` counter to throttle uploads

### RoQInterrupt
- **Signature:** `static void RoQInterrupt(void)`
- **Purpose:** Reads and dispatches one RoQ packet: decodes codebook, VQ video frame, or audio chunk.
- **Side effects:** Calls `decodeCodeBook`, `blitVQQuad32fs` (via function pointer), `RllDecodeMonoToStereo`/`RllDecodeStereoToStereo`, `S_RawSamples`, `S_Update`; updates `RoQPlayed`, `dirty`, `numQuads`

### decodeCodeBook
- **Signature:** `static void decodeCodeBook(byte *input, unsigned short roq_flags)`
- **Purpose:** Rebuilds VQ codebook tables (`vq2`, `vq4`, `vq8`) from raw RoQ codebook packet data for current bpp/scaling mode.
- **Side effects:** Writes into global `vq2`, `vq4`, `vq8` arrays

### RllDecode* (4 functions)
- **Notes:** `RllDecodeMonoToMono`, `RllDecodeMonoToStereo`, `RllDecodeStereoToStereo`, `RllDecodeStereoToMono` — delta-decode RLL audio using `cin.sqrTable`; return sample count

### recurseQuad / setupQuad
- **Notes:** Build the `cin.qStatus` pointer arrays that map RoQ quad-tree cells to pixel offsets in `cin.linbuf`; cached and skipped if geometry is unchanged

## Control Flow Notes
- **Init:** `CIN_PlayCinematic` opens file, calls `initRoQ`→`RoQ_init`, begins streaming
- **Frame update:** Engine calls `SCR_RunCinematic` → `CIN_RunCinematic` each frame; internally loops `RoQInterrupt` until frame count matches elapsed time
- **Render:** `SCR_DrawCinematic` → `CIN_DrawCinematic` submits decoded RGBA buffer to renderer
- **Shutdown:** `CIN_StopCinematic` → `RoQShutdown` closes file, resets `cls.state` if system cinematic

## External Dependencies
- `client.h`: `cls`, `cl`, `uivm`, `re` (renderer), `com_timescale`, `cl_inGameVideo`, `SCR_AdjustFrom640`, `CL_ScaledMilliseconds`
- `snd_local.h`: `s_rawend`, `s_soundtime`, `s_paintedtime`
- Sound: `S_RawSamples`, `S_Update`, `S_StopAllSounds`
- Filesystem: `FS_FOpenFileRead`, `FS_FCloseFile`, `FS_Read`
- Streaming I/O: `Sys_BeginStreamedFile`, `Sys_EndStreamedFile`, `Sys_StreamedRead`
- Renderer: `re.DrawStretchRaw`, `re.UploadCinematic`
- Memory: `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`
- `glConfig.hardwareType`, `glConfig.maxTextureSize` — hardware capability checks
