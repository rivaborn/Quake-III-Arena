# code/win32/win_snd.c

## File Purpose
Windows-specific DirectSound DMA backend for Quake III Arena's audio system. It implements the platform sound device interface (`SNDDMA_*`) using DirectSound COM APIs to drive a looping secondary buffer that the portable mixer writes into.

## Core Responsibilities
- Initialize and tear down a DirectSound device via COM (`CoCreateInstance`)
- Create and configure a secondary DirectSound buffer (hardware-preferred, software fallback)
- Lock/unlock the circular DMA buffer each frame so the mixer can write samples
- Report the current playback position within the DMA ring buffer
- Re-establish the cooperative level when the application window changes focus

## Key Types / Data Structures
None defined in this file; all types come from `snd_local.h` and the Win32 SDK.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `dsound_init` | `qboolean` | static | Tracks whether DirectSound was successfully initialized |
| `sample16` | `int` | static | Shift amount for converting byte position to sample index (0 for 8-bit, 1 for 16-bit) |
| `gSndBufSize` | `DWORD` | static | Actual size in bytes of the secondary DirectSound buffer |
| `locksize` | `DWORD` | static | Number of bytes locked in the most recent `BeginPainting` lock; used by `Submit` |
| `pDS` | `LPDIRECTSOUND` | static | Primary DirectSound object |
| `pDSBuf` | `LPDIRECTSOUNDBUFFER` | static | Secondary (mixing) DirectSound buffer |
| `pDSPBuf` | `LPDIRECTSOUNDBUFFER` | static | Primary DirectSound buffer handle (may equal `pDSBuf`) |
| `hInstDS` | `HINSTANCE` | static | DLL handle for `DSOUND.DLL` (currently unused in load path; legacy) |
| `pDirectSoundCreate` | function pointer | global | Dynamic `DirectSoundCreate` import (macro aliased; left from older load path) |
| `dma` | `dma_t` | extern (global) | Shared DMA descriptor filled by this file; consumed by the portable mixer |

## Key Functions / Methods

### DSoundError
- Signature: `static const char *DSoundError( int error )`
- Purpose: Maps DirectSound HRESULT error codes to human-readable strings for logging.
- Inputs: `error` — HRESULT value
- Outputs/Return: `const char*` string literal
- Side effects: None
- Calls: None
- Notes: Only covers four specific error codes; all others return `"unknown"`.

### SNDDMA_Shutdown
- Signature: `void SNDDMA_Shutdown( void )`
- Purpose: Releases all DirectSound resources, unloads the DLL handle, zeroes `dma`, and calls `CoUninitialize`.
- Inputs: None
- Outputs/Return: void
- Side effects: Releases COM objects `pDSBuf`, `pDSPBuf`, `pDS`; frees `hInstDS`; zeroes `dma`; sets `dsound_init = qfalse`
- Calls: `Com_DPrintf`, `FreeLibrary`, `CoUninitialize`, DirectSound vtbl methods (`Stop`, `Release`, `SetCooperativeLevel`)
- Notes: Guards against double-release of `pDSPBuf` when it equals `pDSBuf`.

### SNDDMA_Init
- Signature: `qboolean SNDDMA_Init(void)`
- Purpose: Entry point called by the portable sound layer to initialize the platform backend; delegates to `SNDDMA_InitDS`.
- Inputs: None
- Outputs/Return: `qtrue` on success, `qfalse` on failure
- Side effects: Calls `CoInitialize(NULL)`; sets `dsound_init = qtrue` on success
- Calls: `CoInitialize`, `SNDDMA_InitDS`, `Com_DPrintf`

### SNDDMA_InitDS
- Signature: `int SNDDMA_InitDS()`
- Purpose: Core DirectSound setup — creates the COM device (DS8 then DS fallback), sets cooperative level, allocates and starts the secondary buffer, and initializes `dma` fields.
- Inputs: None
- Outputs/Return: `1` on success, `qfalse` on failure
- Side effects: Writes `pDS`, `pDSBuf`, `gSndBufSize`, `sample16`, and all `dma` fields; starts looping playback; calls `SNDDMA_BeginPainting`/`SNDDMA_Submit` to clear the buffer
- Calls: `CoCreateInstance`, vtbl `Initialize`, `SetCooperativeLevel`, `CreateSoundBuffer`, `Play`, `GetCaps`, `SNDDMA_BeginPainting`, `SNDDMA_Submit`, `SNDDMA_Shutdown`, `Com_Printf`, `Com_DPrintf`
- Notes: Sample rate is hard-coded to 22050 Hz (commented-out s_khz selection). Tries `DSBCAPS_LOCHARDWARE` first, falls back to `DSBCAPS_LOCSOFTWARE`.

### SNDDMA_GetDMAPos
- Signature: `int SNDDMA_GetDMAPos( void )`
- Purpose: Returns current hardware playback position as a mono-sample offset within the DMA ring buffer.
- Inputs: None
- Outputs/Return: Sample index `[0, dma.samples)`
- Side effects: None
- Calls: `pDSBuf->lpVtbl->GetCurrentPosition`
- Notes: Right-shifts by `sample16` to convert bytes→samples; masks with `dma.samples-1` to wrap.

### SNDDMA_BeginPainting
- Signature: `void SNDDMA_BeginPainting( void )`
- Purpose: Locks the DirectSound buffer and sets `dma.buffer` to the writable pointer; restores/restarts the buffer if lost or stopped.
- Inputs: None
- Outputs/Return: void
- Side effects: Sets `dma.buffer`; modifies `locksize`; may restart playback or call `S_Shutdown` on unrecoverable lock error
- Calls: `pDSBuf->lpVtbl->GetStatus`, `Restore`, `Play`, `Lock`, `S_Shutdown`, `Com_Printf`
- Notes: Retries lost-buffer restore up to 2 times before giving up.

### SNDDMA_Submit
- Signature: `void SNDDMA_Submit( void )`
- Purpose: Unlocks the DirectSound buffer after the mixer has written samples, making the data audible.
- Inputs: None
- Outputs/Return: void
- Side effects: Unlocks `pDSBuf` using `dma.buffer` and `locksize`
- Calls: `pDSBuf->lpVtbl->Unlock`

### SNDDMA_Activate
- Signature: `void SNDDMA_Activate( void )`
- Purpose: Re-asserts `DSSCL_PRIORITY` cooperative level after a window-focus change.
- Inputs: None
- Outputs/Return: void
- Side effects: May call `SNDDMA_Shutdown` on failure
- Calls: `pDS->lpVtbl->SetCooperativeLevel`, `SNDDMA_Shutdown`, `Com_Printf`

## Control Flow Notes
- **Init:** `SNDDMA_Init` → `SNDDMA_InitDS` during engine sound startup.
- **Per-frame:** The portable mixer calls `SNDDMA_BeginPainting` (locks buffer, exposes `dma.buffer`), writes samples, then calls `SNDDMA_Submit` (unlocks). `SNDDMA_GetDMAPos` is polled to determine how much buffer space to fill.
- **Focus change:** `SNDDMA_Activate` is called from the Win32 window procedure on foreground/background transitions.
- **Shutdown:** `SNDDMA_Shutdown` is called by both the portable layer and internally on errors.

## External Dependencies
- `../client/snd_local.h` — `dma_t dma`, `channel_t`, `SNDDMA_*` declarations, `S_Shutdown`
- `win_local.h` — `WinVars_t g_wv` (for `hWnd`), DirectSound/DirectInput version defines, Win32 headers
- `<dsound.h>`, `<windows.h>` — DirectSound COM interfaces
- `Com_Printf`, `Com_DPrintf` — defined in `qcommon`
- `g_wv.hWnd` — window handle from the Win32 platform layer
- `S_Shutdown` — portable sound shutdown, defined in `client/snd_dma.c`
