# code/client/snd_dma.c — Enhanced Analysis

## Architectural Role

`snd_dma.c` is the **orchestration hub** of the Client subsystem's software-mixed audio pipeline. It sits between three distinct layers: the game logic VMs (which submit sound requests via `trap_*` syscalls dispatched through `cl_cgame.c`), the mixing backend (`snd_mix.c` / `snd_mem.c`), and the platform DMA abstraction (`SNDDMA_*` in `win32/win_snd.c` or `unix/linux_snd.c`). It also maintains the `s_rawsamples` ring buffer, which serves as the single shared conduit for all streaming audio — both background music (self-managed here) and cinematic audio fed from `cl_cin.c`. The looping-sound array is sized to `MAX_GENTITIES`, binding the sound system's data model directly to the game entity model defined in `game/g_local.h` and `qcommon`.

## Key Cross-References

### Incoming (who depends on this file)

- **`cl_cgame.c`** — `CL_CgameSystemCalls` dispatches VM trap calls (`CG_S_STARTSOUND`, `CG_S_RESPATIALIZE`, `CG_S_ADDREALLOOPINGSOUND`, `CG_S_CLEARLOOPINGSOUNDS`, `CG_S_REGISTERSOUND`, `CG_S_STARTBACKGROUNDTRACK`, `CG_S_STOPBACKGROUNDTRACK`, `CG_S_UPDATEENTITYPOSITION`) directly to `S_StartSound`, `S_Respatialize`, `S_AddLoopingSound`, `S_RegisterSound`, etc. The cgame VM never calls sound functions directly — always through this syscall boundary.
- **`cl_cin.c`** — The RoQ cinematic player calls `S_RawSamples` to inject decoded audio into the `s_rawsamples` ring buffer each cinematic frame, sharing the same streaming infrastructure as background music.
- **`cl_main.c`** — Calls `S_Init`, `S_Shutdown`, `S_Update` (the per-frame entry point), and `S_StopAllSounds` as part of the outer client frame loop and connection lifecycle.
- **Globals read externally**: `dma`, `s_soundtime`, `s_paintedtime`, `s_rawend`, `s_rawsamples`, `s_channels`, `loop_channels`, `numLoopChannels` are all read by `snd_mix.c` (`S_PaintChannels`) to drive the actual sample mixing.

### Outgoing (what this file depends on)

- **`snd_mix.c`** — `S_PaintChannels(s_paintedtime, endtime)` is the downstream mixing call; everything in `snd_dma.c` exists to feed it properly aligned work.
- **`snd_mem.c`** — `S_LoadSound`, `SND_malloc/free`, `SND_setup`, `S_memoryLoad` for asset loading and buffer chain management.
- **Platform layer** (`win32/win_snd.c` or `unix/linux_snd.c`) — `SNDDMA_Init`, `SNDDMA_Shutdown`, `SNDDMA_GetDMAPos`, `SNDDMA_BeginPainting`, `SNDDMA_Submit` provide the hardware abstraction; `snd_dma.c` has zero platform-specific code itself.
- **OS streaming I/O** — `Sys_BeginStreamedFile`, `Sys_StreamedRead`, `Sys_EndStreamedFile` for background WAV streaming (these are defined in platform layers, not in `qcommon`).
- **`qcommon`** — `Cvar_Get`, `Cmd_AddCommand`, `Com_Milliseconds`, `FS_FOpenFileRead/FCloseFile/Read` for config, command registration, timing, and filesystem access. Also reads `cls.framecount` (from `client.h`) for Doppler per-frame tracking.
- **`q_math.c` / `q_shared.c`** — `VectorSubtract`, `VectorNormalize`, `VectorRotate`, `DistanceSquared`, `DotProduct` for all spatialization math.

## Design Patterns & Rationale

- **Intrusive free-list for channel pool**: `channel_t *freelist` embeds the next pointer at offset 0 of `channel_t` itself (`*(channel_t **)v`). This avoids a separate allocation structure and was idiomatic in late-90s game engines — zero per-node overhead, cache-friendly for small pools.
- **Two-phase channel model**: One-shot channels (`s_channels`) are persistent across frames; loop channels (`loop_channels`) are rebuilt every frame from `loopSounds[]`. This avoids the complexity of tracking loop-sound state across entity repositioning — simpler than merging persistent state.
- **Ring buffer with a single monotonically increasing `s_rawend` cursor**: The raw sample buffer (`s_rawsamples[MAX_RAW_SAMPLES]`) is indexed modulo `MAX_RAW_SAMPLES`. Both background music and cinematic audio advance this single pointer, meaning only one streaming source can be active at a time without collision. This is a deliberate simplification — the engine doesn't support simultaneous music + cinematic audio.
- **Platform abstraction via `SNDDMA_*` vtable pattern (without vtable)**: The DMA layer is a set of globally-linked platform functions rather than a vtable. The tradeoff: no dynamic dispatch cost, but requires relinking for each platform. This matches the era's preference for static platform selection at compile time.
- **Lazy eviction** (`S_FreeOldestSound`): Memory pressure is handled by evicting the LRU sfx only when a new buffer is needed. Modern engines typically use explicit asset budgets or streaming with priority queues.

## Data Flow Through This File

```
cgame VM trap call
  → cl_cgame.c CL_CgameSystemCalls
    → S_StartSound / S_AddLoopingSound           [writes s_channels / loopSounds[]]
    → S_Respatialize                             [updates listener_* globals]
      → S_AddLoopSounds                          [builds loop_channels[] from loopSounds[]]
      → S_SpatializeOrigin (per channel)         [left_vol/right_vol computed]

cl_main.c per-frame
  → S_Update
    → S_UpdateBackgroundTrack                    [Sys_StreamedRead → S_RawSamples → s_rawsamples[]]
    → S_Update_
      → S_GetSoundtime                           [SNDDMA_GetDMAPos → s_soundtime]
      → SNDDMA_BeginPainting                     [lock DMA buffer]
      → S_PaintChannels(s_paintedtime, endtime)  [snd_mix.c reads s_channels, loop_channels, s_rawsamples]
      → SNDDMA_Submit                            [unlock + submit to hardware]

cl_cin.c cinematic frame
  → S_RawSamples                                 [decoded PCM → s_rawsamples[]]
```

Key state transitions: `s_soundMuted` toggled by `S_DisableSounds` (hunk clear) / `S_BeginRegistration` (map load); `s_soundStarted` guards all operations.

## Learning Notes

- **Doppler without DSP**: Doppler effect is approximated by comparing entity velocity against a speed threshold and adjusting playback rate on the loop channel. This is entirely in the integer domain — no pitch-shift filter, just sample-step scaling in the mixer. Modern engines use pitch-shifting via resampling in a proper DSP graph.
- **No audio graph / bus system**: All channels mix directly to the DMA buffer with no intermediate buses, aux sends, or reverb. Every effect (spatialization, doppler, volume attenuation) is computed per-channel in-line. This is the defining tradeoff of late-90s software mixers — simplicity at the cost of DSP flexibility.
- **`MAX_GENTITIES` coupling**: `loopSounds[MAX_GENTITIES]` directly mirrors the game's entity array size. This is a flat, index-parallel design — zero pointer chasing, but tight coupling between sound and game systems. Modern engines use component systems or event buses to decouple these.
- **Compressed audio is stubbed out**: `compressed = qfalse` is unconditionally forced in `S_RegisterSound`, though the `sfx_t.soundCompressed` field and ADPCM/wavelet code exist in `snd_adpcm.c` / `snd_wavelet.c`. This reveals a feature that was planned but disabled before ship — developers studying the code should not expect wavelet compression to function.
- **The hash table is open-addressed via chaining, not probing**: `sfxHash[]` holds singly-linked `sfx_t` chains. With only 128 buckets and up to 4096 sounds, chains can grow long — this was acceptable because sound lookup is infrequent (registration time only, not per-frame).

## Potential Issues

- **Single raw buffer for all streaming sources**: `s_rawsamples` is shared between background music and cinematics with no arbitration. Simultaneous playback of both will corrupt the buffer — by inspection this appears to be prevented by convention (cinematic stops music before playing), but there is no enforced invariant.
- **Channel-stealing by `allocTime`**: The oldest channel is stolen when `freelist` is empty. Under heavy load with many simultaneous sounds, recently-started sounds could be immediately evicted if the allocation timestamp resolution (`Com_Milliseconds`) isn't fine enough — all sounds starting in the same millisecond have equal `allocTime` and stealing becomes arbitrary.
- **`s_numSfx` scan on allocation**: `S_FindName` does a linear scan through `s_knownSfx` to find a free slot before checking the hash table. With up to 4096 entries this is O(n) per registration — not a runtime issue (registration is level-load-time only) but architecturally inelegant.
