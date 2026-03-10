# code/null/null_snddma.c

## File Purpose
Provides a null (no-op) implementation of the platform-specific sound DMA driver interface. It exists to allow Quake III Arena to compile and run without any audio hardware or audio subsystem, returning safe default values for all sound queries.

## Core Responsibilities
- Stub out `SNDDMA_*` lifecycle functions so the portable sound mixer has valid symbols to call
- Stub out higher-level `S_*` sound API functions to prevent crashes in headless or null-platform builds
- Return `qfalse`/`0` from all init/query functions to signal audio is non-functional

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### SNDDMA_Init
- Signature: `qboolean SNDDMA_Init(void)`
- Purpose: Initializes the sound DMA driver; always reports failure.
- Inputs: None
- Outputs/Return: `qfalse` — signals to the engine that no audio device is available
- Side effects: None
- Calls: None
- Notes: The portable sound system (`snd_dma.c`) checks this return value and disables audio entirely if `qfalse`.

### SNDDMA_GetDMAPos
- Signature: `int SNDDMA_GetDMAPos(void)`
- Purpose: Returns the current DMA write position in the audio buffer.
- Inputs: None
- Outputs/Return: `0`
- Side effects: None
- Calls: None
- Notes: A real implementation returns the hardware playback cursor offset; returning `0` here is safe since audio is disabled.

### SNDDMA_Shutdown
- Signature: `void SNDDMA_Shutdown(void)`
- Purpose: Shuts down the sound DMA driver.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None

### SNDDMA_BeginPainting
- Signature: `void SNDDMA_BeginPainting(void)`
- Purpose: Called before the portable mixer paints audio samples into the DMA buffer.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None

### SNDDMA_Submit
- Signature: `void SNDDMA_Submit(void)`
- Purpose: Submits/flushes the painted DMA buffer to the audio hardware.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None

### S_RegisterSound
- Signature: `sfxHandle_t S_RegisterSound(const char *name, qboolean compressed)`
- Purpose: Registers a sound asset; always returns an invalid handle.
- Inputs: `name` — sound asset path; `compressed` — whether ADPCM-compressed
- Outputs/Return: `0` (null handle)
- Side effects: None
- Calls: None
- Notes: Comment indicates the `compressed` boolean was added in bk001119 to match `snd_public.h`.

### S_StartLocalSound
- Signature: `void S_StartLocalSound(sfxHandle_t sfxHandle, int channelNum)`
- Purpose: Plays a local (non-spatialized) sound; silently discarded.
- Inputs: `sfxHandle` — handle to registered sound; `channelNum` — channel index
- Outputs/Return: None
- Side effects: None
- Calls: None

### S_ClearSoundBuffer
- Signature: `void S_ClearSoundBuffer(void)`
- Purpose: Clears the sound mixing buffer; no-op here.
- Inputs: None
- Outputs/Return: None
- Side effects: None
- Calls: None

## Control Flow Notes
This file is entirely passive — no init/frame/shutdown sequencing occurs. It is linked in place of a real platform sound driver (e.g., `linux_snd.c`, `win_snd.c`) for null/dedicated-server builds. The engine calls `SNDDMA_Init` during startup; receiving `qfalse` prevents the portable mixer from ever calling `SNDDMA_BeginPainting` or `SNDDMA_Submit` in the frame loop.

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h` (for `qboolean`, `sfxHandle_t`) and `snd_public.h` (for the sound API contract)
- `sfxHandle_t` — typedef defined in `snd_public.h` (defined elsewhere)
- `qboolean`, `qfalse` — defined in `q_shared.h` (defined elsewhere)
