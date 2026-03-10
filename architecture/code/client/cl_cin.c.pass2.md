# code/client/cl_cin.c — Enhanced Analysis

## Architectural Role

This file implements the **complete RoQ video cinematic subsystem** within the Client layer, operating as a specialized producer for both the renderer and sound subsystems. It occupies a unique architectural position: orthogonal to game/cgame logic (never calls back to VMs), yet tightly coupled to frame/audio timing and downstream playback (renderer for 2D drawing, sound system for PCM mixing). Cinematics are initialized from `CIN_PlayCinematic` (called by `SCR_RunCinematic` in `cl_scrn.c` or by direct client code) and updated each frame via `CIN_RunCinematic`, creating a **side-channel data pipeline** that bypasses the normal entity/snapshot flow.

## Key Cross-References

### Incoming (who depends on this file)
- **`cl_scrn.c`** → calls `SCR_RunCinematic` / `SCR_DrawCinematic` (not defined here, but orchestrates this file)
- **`client.h`** → exports `CIN_PlayCinematic`, `CIN_RunCinematic`, `CIN_StopCinematic`, `CIN_DrawCinematic`, `CIN_UploadCinematic`, `CIN_CloseAllVideos`
- **Console commands** → e.g., `cin`, `cinematic` commands likely route through client console system
- **UI VM** (`code/q3_ui`, `code/ui`) → may call into cgame/engine to play cinematics, which triggers this subsystem

### Outgoing (what this file depends on)
- **Renderer** → `re.DrawStretchRaw`, `re.UploadCinematic` (submit decoded RGBA and optional texture upload)
- **Sound system** → `S_RawSamples`, `S_Update`, `S_StopAllSounds` (feed RLL-decoded PCM; manage audio during cinematic)
- **Filesystem** → `FS_FOpenFileRead`, `FS_FCloseFile`, `FS_Read` (load `.roq` files from pak/directory)
- **Streaming I/O** → `Sys_BeginStreamedFile`, `Sys_EndStreamedFile`, `Sys_StreamedRead` (background I/O scheduling)
- **Memory** → `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory` (large temporary buffers for decode workspace)
- **Client state** → reads/writes `cls.state` (transition to `CA_CINEMATIC`), `cl_inGameVideo` cvar
- **Engine globals** → `glConfig.hardwareType`, `glConfig.maxTextureSize` (hardware-specific downsampling)

## Design Patterns & Rationale

**1. Standalone Codec Handler**  
RoQ is a **real-time VQ-based video codec** (not DCT-like MPEG). The file encapsulates the entire decode pipeline: file parsing, codebook decompression, quad-tree blitting, and color space conversion. This pattern avoids polluting the general-purpose rendering pipeline with codec-specific logic.

**2. Function Pointer Dispatch for VQ Blitting**  
The `cin_cache` struct holds `VQ0`, `VQ1`, `VQNormal`, `VQBuffer` function pointers, allowing **dynamic selection** of blit routines based on color depth (16-bit vs 32-bit) and scaling mode. This avoids branching inside the inner loop during frame decoding.

**3. Precomputed YUV→RGB Lookup Tables**  
Static tables (`ROQ_YY_tab`, `ROQ_UB_tab`, etc.) and VQ codebook cache (`vq2`, `vq4`, `vq8`) are initialized once, then reused per-frame. This trades **memory (O(256×4 longwords + 256×16×4 shorts))** for **CPU (no YUV math per pixel)** — a classic video decoder optimization.

**4. Quad-Tree Geometry Caching via `qStatus` Pointers**  
The `cin.qStatus[2][32768]` array is built once per geometry change (via `setupQuad`/`recurseQuad`), mapping VQ quad-tree cells to pixel offsets in the line buffer. This avoids recomputing pointer arithmetic every frame.

**5. Dual-Time Synchronization**  
Frames are advanced to match **wall-clock time** (from `CL_ScaledMilliseconds`) AND **audio time** (from `s_soundtime`/`s_paintedtime`). This ensures cinematics stay synchronized with RLL-decoded audio even if the frame rate varies or the game is paused.

**6. Hardware Adaptation via Downsampling**  
Old hardware (Voodoo, Rage Pro) lacks large texture support. The renderer's `SCR_DrawCinematic` may request 256×256 downsampling. This is handled at blit time via a temporary hunk allocation — a pragmatic adaptation pattern.

## Data Flow Through This File

```
                        File I/O                       Frame Update Loop (per frame)
        .roq file ──────────────────→ [FILE BUFFER]
                                             │
                      [RoQInterrupt in loop while frame count < elapsed time]
                             │
                    ┌─────────┼─────────┐
                    ↓         ↓         ↓
            CODEBOOK   VQ FRAME    AUDIO PACKET
                    │         │         │
        decodeCodeBook    blitVQQuad*   RllDecode*
                    │         │         │
              vq2/vq4/vq8  [linbuf]   [s_soundtime feed]
                              │         │
                    ┌─────────┴─────────┘
                    ↓
        [Frame complete: mark dirty=1, update RoQPlayed]
                    │
        ┌───────────┴───────────┐
        ↓                       ↓
   CIN_DrawCinematic     CIN_UploadCinematic
   (screen blit)        (in-game texture)
        │                       │
   re.DrawStretchRaw      re.UploadCinematic
        │                       │
    [RENDERER]            [in-game surfaces]
```

**Key state transitions:**
- `FMV_IDLE` → `FMV_PLAY` (on `CIN_PlayCinematic`)
- `FMV_PLAY` → `FMV_LOOPED` (on loop boundary, if `looping=1`)
- `FMV_PLAY` → `FMV_EOF` (on file end, if `holdAtEnd=0`)
- `FMV_EOF` / `FMV_LOOPED` ← `CIN_StopCinematic` forces `FMV_EOF`

## Learning Notes

**1. RoQ: A VQ-Based Ancestor of Modern Codecs**  
Unlike DCT-based schemes (MPEG, H.264), RoQ uses **Vector Quantization**: the frame is recursively partitioned into 2×2/4×4/8×8 quads, each of which is either a **copy of a codebook entry** or a **recursion hint**. This made RoQ decodable on mid-1990s hardware (fixed lookup + memcpy pattern). Modern engines use H.264/VP9 (better compression, streaming-friendly). Studying RoQ shows **why** the shift happened: limited compression ratio, frame-type dependencies, inflexible error resilience.

**2. Dual-Time Audio/Video Sync**  
The pattern of syncing to **both wall-clock and audio buffer fill** is crucial for **streaming in real-time**. If you only sync to wall-clock, audio may stutter if the game frame rate drops. If you only sync to audio, the video may drift if audio underruns. This file balances both via `CL_ScaledMilliseconds` (account for `com_timescale`) and `s_soundtime` (sound buffer position).

**3. Streaming I/O as a Separate Concern**  
The use of `Sys_BeginStreamedFile` / `Sys_StreamedRead` (as opposed to synchronous `FS_Read`) shows that **platform layers can offer background I/O scheduling**. This file decodes only what it reads; the platform may asynchronously fill the buffer during idle time. A lesson for modern engines: separate I/O scheduling from decode logic.

**4. Function Pointers for Code-Path Selection**  
Rather than branching on color depth inside the blit loop, this file uses **vtable-style function pointers** (`VQ0`, `VQ1`, etc. set once at init). This is an early instance of **avoiding runtime polymorphism overhead** in hot paths — a pattern that predates C++ vtables and is still relevant for SIMD/JIT specialization.

**5. Idiomatic to Q3A Era: Direct Codec in Engine**  
Modern engines abstract codecs behind OS/middleware layers (Windows Media Foundation, macOS AVFoundation, ffmpeg, libvpx). Q3A's approach — embed RoQ decode directly in the engine — meant **binary compatibility and control** at the cost of **maintenance burden**. This reflects the 1999 era's priorities.

## Potential Issues

1. **Single Global `cin` Workspace**  
   All 16 handles share a single `cinematics_t` decode buffer (`cin.linbuf`, `cin.sqrTable`, `cin.qStatus`). If two cinematics are playing simultaneously and `currentHandle` is switched mid-decode, **state corruption is possible**. The code assumes **single-active-handle at runtime**, which is enforced by the client loop but not by API contract.

2. **No Explicit Error Recovery**  
   Malformed RoQ files (truncated packets, invalid quad codes, bad codebook data) may trigger `Com_Error` or cause silent corruption in the line buffer. There's no checksum/CRC validation of decoded frames before submission to the renderer.

3. **Hardware Downsampling Hunk Leaks**  
   `CIN_DrawCinematic` allocates `Hunk_AllocateTempMemory` for 256×256 downsampling but relies on the same call to free it. If an error path is taken mid-draw, the hunk may leak. Modern code would use RAII or explicit cleanup.

4. **No Streaming Pause/Resume**  
   If `Sys_StreamedRead` returns 0 bytes (slow I/O), the file read may block or stall. There's no backpressure mechanism; if the platform can't keep up, the decode may fall behind and cause audio/video desync.

---
