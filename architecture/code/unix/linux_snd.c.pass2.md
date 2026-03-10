# code/unix/linux_snd.c — Enhanced Analysis

## Architectural Role

This file implements the platform-specific sound device driver interface (`SNDDMA_*` vtable) for the client audio subsystem. The portable mixer (`snd_dma.c`, `snd_mix.c`) consumes these functions to drive DMA-based audio output on Linux/FreeBSD. It is one of four equivalent platform layers (`unix/linux_snd.c`, `win32/win_snd.c`, `macosx/macosx_snddma.m`, `null/null_snddma.c`), selected at link time. The architecture isolates portable mixing from hardware abstraction—a clean boundary that enables Q3A to target multiple sound APIs (OSS, DirectSound, CoreAudio) with identical game logic.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/client/snd_dma.c`** — Calls `SNDDMA_Init` once during `S_Init`, then `SNDDMA_GetDMAPos` every frame in `S_Update`; paints audio into the buffer between `SNDDMA_BeginPainting` (no-op) and `SNDDMA_Submit` (no-op)
- **Client frame loop** (`code/client/cl_main.c`) — Indirectly invokes the above via `S_Frame`

### Outgoing (what this file depends on)
- **`dma` global** (external, defined in `code/client/snd_dma.c`) — Shared DMA state: `buffer`, `samplebits`, `speed`, `channels`, `samples`, `submission_chunk`
- **`saved_euid`** (extern from `code/unix/unix_main.c`) — Preserved effective UID for privileged device access
- **`Cvar_Get`, `Com_Printf`, `Com_Memset`** from `code/qcommon` — Console variable registration, error logging, fallback memory ops
- **System headers** — `<linux/soundcard.h>` (Linux) or `<sys/soundcard.h>` (FreeBSD) for OSS ioctl definitions

## Design Patterns & Rationale

**Privilege Escalation via seteuid**: Opens `/dev/dsp` with elevated effective UID (`saved_euid`), then drops back to unprivileged (`getuid()`). This follows Unix capability-dropping patterns, though modern systems would use `/dev/snd/*` with group permissions.

**Fallback Chains**:
- Sample rate: `tryrates[]` array tested in order (22050, 11025, 44100, 48000, 8000). Auto-detection tolerates hardware constraints.
- mmap protection: First tries `PROT_READ|PROT_WRITE`; on failure, downgrades to `PROT_WRITE` and enables custom `Snd_Memset`. This reflects a known glibc bug (bugzilla #371) on ALSA-based Linux systems where kernel DMA buffers don't support read protection.
- Sample format: Probes `AFMT_S16_LE`, falls back to `AFMT_U8`.

**Polling-Based DMA Model**: Unlike interrupt-driven sound systems, the mixer polls `SNDDMA_GetDMAPos` every frame to determine the hardware read pointer. The engine then paints ahead of that pointer into the mmap'd ring buffer. Kernel DMA reads directly from the buffer—no explicit `write()` calls needed.

**Ring-Buffer Semantics**: The kernel divides the mmap'd region into `fragstotal` fragments of `fragsize` bytes each. Engine and DMA engine maintain independent read/write cursors in the same circular buffer. No explicit synchronization primitives—just position arithmetic.

## Data Flow Through This File

1. **Initialization (one-time, `SNDDMA_Init`)**:
   - Register CVARs if not already done (`snddevice` guard)
   - Open `/dev/dsp` with privilege escalation
   - Query hardware capabilities (trigger support, mmap support)
   - Negotiate stereo/mono via `SNDCTL_DSP_STEREO`
   - Negotiate sample rate via `SNDCTL_DSP_SPEED` (tries `tryrates[]`)
   - Negotiate sample format via `SNDCTL_DSP_SETFMT`
   - Query buffer layout via `SNDCTL_DSP_GETOSPACE` → compute `dma.samples`
   - mmap the kernel DMA buffer into `dma.buffer`
   - Arm trigger with `SNDCTL_DSP_SETTRIGGER` → DMA engine begins reading

2. **Per-Frame (mixer loop in `snd_dma.c`)**:
   - Call `SNDDMA_GetDMAPos` → reads current DMA pointer via `SNDCTL_DSP_GETOPTR`
   - Mixer calculates safe paint region relative to DMA read cursor
   - Application writes audio samples directly into `dma.buffer` (mmap'd kernel memory)
   - No explicit flush—kernel's DMA engine reads continuously from the buffer

3. **Shutdown (never explicitly called; implicit at process exit)**:
   - `SNDDMA_Shutdown` is a no-op
   - File descriptor and mmap remain open
   - Kernel auto-closes at process termination

## Learning Notes

**OSS Era (2005)**: This code targets OSS (Open Sound System), the pre-ALSA standard. OSS is simpler (fewer ioctl types, direct device paths like `/dev/dsp`) but less flexible. By 2025, ALSA dominates Linux audio, and OSS compatibility is often provided by wrapper libraries. The code's survival in the Q3 codebase reflects either a "still works" principle or active use on BSD systems (which continue supporting OSS natively).

**Memory-Mapped DMA vs. Modern Approaches**:
- **Q3A approach**: mmap kernel ring buffer, write directly, let kernel DMA read. Low-latency, zero-copy.
- **Modern systems**: PipeWire/PulseAudio software mixing, ALSA dmix, or exclusive hardware access. These hide the ring buffer behind a higher-level abstraction.

**Privilege Management Anti-pattern**: The `seteuid` dance is characteristic of early-2000s game engines. Modern systems enforce audio device permissions via group membership (`/dev/snd/`, `/dev/audio`) rather than SUID binaries, reducing attack surface.

**Workaround Archaeology**: The `use_custom_memset` flag and Snd_Memset fallback encode a real bug encountered in glibc on some Linux distributions. The mmap read-protection issue suggests an ALSA kernel module or libc interaction that prevented safe simultaneous read+write access. This kind of defensive code is idiomatic for cross-platform game engines targeting hardware with variable OS configurations.

## Potential Issues

1. **No Transient Error Recovery**: `SNDDMA_GetDMAPos` immediately closes the device on `ioctl` failure (perror, Com_Printf, close, flag snd_inited=0). A momentary device stall (blocked task, transient DMA glitch) crashes audio. Production systems would implement retry logic or watchdog timers.

2. **CVAR Registration Race**: The `if (!snddevice)` guard assumes single-threaded init. On concurrent calls (hypothetically), `Cvar_Get` could execute twice or `snddevice` could be garbage-read during registration. A static `int inited` flag would be safer.

3. **Incomplete mmap Cleanup**: `SNDDMA_Shutdown` is a no-op; `munmap(dma.buffer, ...)` is never called. Kernel reclaims the mapping at process exit, but leaving it unexamplary. In a long-running server, repeated reinit would leak file descriptors.

4. **Hard-Coded `/dev/dsp` Fragility**: Default device path is unlikely to exist on modern systems or ALSA-only machines. No fallback to `/dev/audio`, `/dev/snd/pcmC0D0p`, or alternative devices. Silent failure on missing device.

5. **Memset Workaround Incompleteness**: Custom `Snd_Memset` only activates if `PROT_WRITE|PROT_READ` mmap fails. If the same glibc bug affects write-only mmap, or if some other code path calls `Com_Memset` directly on the buffer, the bug persists. The fix is defensive but not comprehensive.

6. **No Format Negotiation Feedback**: Driver silently selects 8-bit or 16-bit if the requested format is unavailable. No warning to user that audio quality degraded. A `Com_Printf` would be appropriate.
