# code/client/snd_public.h — Enhanced Analysis

## Architectural Role

This file defines the boundary between **event-driven sound consumers** (cgame VM, client UI, cinematics) and the **server-managed audio pipeline**. As the sole public interface to the software-mixed sound system, it abstracts away the DMA layer, channel allocation, and real-time mixing from the 3D game world. The header's design enforces a strict per-frame protocol: positional entities are registered, sounds are queued, then `S_Update` atomically mixes and submits to hardware.

## Key Cross-References

### Incoming (who depends on this file)
- **cgame VM** (`code/cgame/cg_event.c`): Fires `EV_*` events that translate into `S_StartSound` calls with weapon, damage, footstep, and environmental audio
- **Client layer** (`code/client/cl_*.c`): Drives `S_Update` each frame from the main loop; calls `S_Respatialize` with the local player's position and view axes
- **Server/Game VM** (indirectly): `trap_StartSound` in `code/game/g_*.c` sends events through the network snapshot → cgame → sound system
- **UI VMs** (`code/q3_ui`, `code/ui`): Menu sounds via `S_StartLocalSound`

### Outgoing (what this file depends on)
- **Platform DMA** (`win32/win_snd.c`, `unix/linux_snd.c`): `SNDDMA_Activate` reactivates audio on focus restore; DMA buffer submission during `S_Update`
- **Virtual filesystem** (`code/qcommon/files.c`): `S_RegisterSound` loads `.wav` and `.ogg` assets via `FS_ReadFile`
- **Memory system** (`code/qcommon/cmd.c` and zone allocator): `S_BeginRegistration` / asset caching
- **Collision/BSP** (implicit): Sound propagation may use PVS or traces (not visible in this header, but referenced by implementation)
- **Globals** (implicit): Entity position cache shared with entity system

## Design Patterns & Rationale

**Frame-locked looping protocol**: The sequence `S_ClearLoopingSounds` → `S_AddLoopingSound` (per active entity) → `S_Update` ensures looping sounds track dynamic entity motion without manual position updates. This prevents "stale" loops and avoids per-update position queries.

**Handle-based asset system**: `S_RegisterSound` returns an opaque `sfxHandle_t` handle, never failing even for missing assets. This pattern (common in late-2000s engines) avoids filesystem checks during gameplay and simplifies error handling in hot paths.

**Separated world and local playback**: `S_StartSound` (3D spatialized) vs. `S_StartLocalSound` (2D UI) reflects the engine's clear separation between game-world audio and non-diegetic UI feedback.

**Raw PCM injection** (`S_RawSamples`): Bypasses the registration/handle system entirely for streaming VoIP and cinematic audio—indicative of the era's need to support RoQ video with synchronized low-latency audio.

## Data Flow Through This File

1. **Registration phase**: cgame init calls `S_BeginRegistration`, then `S_RegisterSound` for all level-specific audio (weapons, impacts, voices). Handles are cached locally.
2. **Per-frame event loop**: 
   - cgame consumes server snapshot → fires `EV_*` events
   - Event handlers call `S_StartSound` with entity origin/number or `S_StartLocalSound`
   - Looping sounds are cleared and re-added for all active entities
   - `S_Respatialize` recomputes listener-relative volumes
   - `S_Update` mixes all channels, applies 3D panning, and submits to DMA buffer
3. **Streaming**: `S_StartBackgroundTrack` queues intro+loop for background music; `S_UpdateBackgroundTrack` refills streaming buffers each frame

## Learning Notes

**Idiomatic to this era**: The software-mixed, handle-based approach reflects pre-DirectX 3D Audio and pre-OpenAL dominance (early 2000s). Modern engines (Wwise, FMOD, HiFi Audio) are middleware-based and use hardware acceleration or multi-threaded mixing.

**Key architectural assumption**: Entity-to-position coupling is **external**—the sound system queries entity state at respatialize time, never owns it. This enforces clear data ownership and avoids circular dependencies between game state and audio subsystems.

**Frame synchronization**: The protocol (clear → add → respatialize → update) is entirely synchronous and single-threaded, relying on the client's per-frame cadence. There's no decoupling for variable-rate audio; audio and game run at the same tick.

## Potential Issues

- **Silent failure on registration**: `S_RegisterSound` returning a placeholder for missing files masks asset pipeline errors silently. Developers must rely on logs, not API feedback.
- **No voice-channel preemption policy visible**: When all channels are full, the header doesn't document which sound is dropped (oldest? quietest?). Implementation detail leaks into API design.
- **Listener-only spatialization**: The API assumes a single listener (local player). Network traces, soundscapes, and multi-listener VR are not addressable through this interface.
