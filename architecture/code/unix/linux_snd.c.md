# code/unix/linux_snd.c

## File Purpose
Linux/FreeBSD platform-specific DMA sound driver for Quake III Arena. It opens the OSS `/dev/dsp` device, configures it for mmap-based DMA audio output, and implements the `SNDDMA_*` interface consumed by the portable sound mixing layer.

## Core Responsibilities
- Register and validate sound CVARs (`sndbits`, `sndspeed`, `sndchannels`, `snddevice`)
- Open the OSS sound device with privilege escalation (`seteuid`)
- Negotiate sample format, rate, and channel count via `ioctl`
- Memory-map the DMA ring buffer into `dma.buffer`
- Arm the DSP trigger to begin output
- Query the current playback pointer (`GETOPTR`) each frame
- Work around a glibc `memset` bug via a custom `Snd_Memset` fallback

## Key Types / Data Structures
None defined in this file; all types are from included headers.

| Name | Kind | Purpose |
|------|------|---------|
| `dma_t` | struct (external) | Holds DMA config: `buffer`, `samplebits`, `speed`, `channels`, `samples`, `submission_chunk` |
| `cvar_t` | struct (external) | Engine console variable holding name/value pairs |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `audio_fd` | `int` | global | File descriptor for the open OSS `/dev/dsp` device |
| `snd_inited` | `int` | global | Guard flag; non-zero if DMA is fully initialized |
| `sndbits` | `cvar_t *` | global | Console var: bits per sample (8 or 16) |
| `sndspeed` | `cvar_t *` | global | Console var: sample rate (0 = auto-detect) |
| `sndchannels` | `cvar_t *` | global | Console var: number of channels (1 or 2) |
| `snddevice` | `cvar_t *` | global | Console var: device path, default `/dev/dsp` |
| `tryrates` | `static int[]` | static | Ordered list of fallback sample rates to probe |
| `use_custom_memset` | `static qboolean` | static | Set when `mmap(PROT_WRITE\|PROT_READ)` fails; triggers word-loop `Snd_Memset` |

## Key Functions / Methods

### Snd_Memset
- **Signature:** `void Snd_Memset(void *dest, const int val, const size_t count)`
- **Purpose:** glibc `memset` bug workaround (bugzilla #371). Uses a plain `int`-stride write loop when `use_custom_memset` is true; falls through to `Com_Memset` otherwise.
- **Inputs:** destination pointer, fill value, byte count
- **Outputs/Return:** void
- **Side effects:** Writes to `dest`.
- **Calls:** `Com_Memset`
- **Notes:** Only active on Linux when `PROT_READ` mmap fails. Declared in `q_shared.h` for Linux builds.

### SNDDMA_Init
- **Signature:** `qboolean SNDDMA_Init(void)`
- **Purpose:** Full initialization of OSS DMA audio: opens device, negotiates format/rate/channels, mmaps the DMA buffer, and starts output trigger.
- **Inputs:** none (reads CVARs)
- **Outputs/Return:** `qtrue` (1) on success, `qfalse` (0) on any failure.
- **Side effects:** Sets `audio_fd`, `snd_inited`, `dma.*`, `use_custom_memset`. Calls `seteuid`/`getuid`. Writes to `perror`/`Com_Printf` on error. Closes `audio_fd` on any failure.
- **Calls:** `Cvar_Get`, `seteuid`, `getuid`, `open`, `close`, `ioctl` (GETCAPS, GETFMTS, SPEED, STEREO, SETFMT, GETOSPACE, SETTRIGGER), `mmap`, `perror`, `Com_Printf`
- **Notes:** `snddevice` acts as a once-only init guard — CVARs are only registered on the first call. Rate auto-detection iterates `tryrates[]`. Falls back to write-only mmap if read+write mmap fails, enabling `use_custom_memset`. `saved_euid` is declared `extern` and defined elsewhere (unix_main).

### SNDDMA_GetDMAPos
- **Signature:** `int SNDDMA_GetDMAPos(void)`
- **Purpose:** Returns the current hardware playback position in samples, used by the mixer to determine how far ahead to paint.
- **Inputs:** none
- **Outputs/Return:** Sample offset (bytes / (samplebits/8)); 0 if not initialized or on error.
- **Side effects:** On `ioctl` failure: prints error, closes `audio_fd`, clears `snd_inited`.
- **Calls:** `ioctl` (GETOPTR), `perror`, `Com_Printf`, `close`

### SNDDMA_Shutdown / SNDDMA_Submit / SNDDMA_BeginPainting
- All three are **no-ops** (empty function bodies). Submit and BeginPainting are vacuous because the mmap DMA model requires no explicit transfer; the kernel reads directly from the mapped buffer.

## Control Flow Notes
- **Init:** `SNDDMA_Init` is called once during sound system startup (`S_Init` in `snd_dma.c`).
- **Per-frame:** `SNDDMA_BeginPainting` (no-op) → mixer paints into `dma.buffer` → `SNDDMA_Submit` (no-op). `SNDDMA_GetDMAPos` is polled each frame to track the write cursor.
- **Shutdown:** `SNDDMA_Shutdown` is a no-op; the device is only closed on error paths inside `SNDDMA_Init`/`SNDDMA_GetDMAPos`.

## External Dependencies
- **System headers:** `<unistd.h>`, `<fcntl.h>`, `<sys/ioctl.h>`, `<sys/mman.h>`, `<linux/soundcard.h>` (Linux) / `<sys/soundcard.h>` (FreeBSD)
- **Local headers:** `../game/q_shared.h`, `../client/snd_local.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`, global) — `snd_dma.c`
  - `saved_euid` (`uid_t`) — `unix_main.c`
  - `Cvar_Get`, `Com_Printf`, `Com_Memset` — engine core
