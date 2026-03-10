# code/cgame/cg_playerstate.c — Enhanced Analysis

## Architectural Role

This file is the **centerpiece of cgame's event/feedback response layer**, sitting at the critical junction between network snapshots and local-only client feedback. It orchestrates the transition from one authoritative `playerState_t` to the next, firing all associated events (network events, predicted events, damage feedback, announcer sounds, UI state changes) in the correct order. While it doesn't modify game logic or simulation, it bridges the gap between server authority and client perception—essential for making networked combat feel responsive.

## Key Cross-References

### Incoming (who depends on this file)
- **`cg_snapshot.c`:** Calls `CG_TransitionPlayerState` after advancing the snapshot buffer; drives the per-frame event loop during normal gameplay
- **`cg_predict.c`:** Calls `CG_CheckChangedPredictableEvents` after the prediction loop to reconcile predicted events against server-authoritative events
- **Render/draw code** (e.g., `cg_draw.c`): Reads `cg.damageX/Y/Value`, `cg.lowAmmoWarning`, `cg.timelimitWarnings` to render HUD elements and view-kick effects
- **Animation/particle code** (e.g., `cg_localents.c`): Consumes event-triggered effects pushed by `CG_EntityEvent` calls

### Outgoing (what this file depends on)
- **Entity event system** (`cg_event.c::CG_EntityEvent`): Routes playerstate events into the effect/sound/particle dispatchers
- **Sound subsystem** (`trap_S_StartLocalSound`, `CG_AddBufferedSound`, `cg_consolecmds.c`): Plays ammo warnings, hit feedback, announcer callouts, music cues
- **Pain/damage system** (`CG_PainEvent`): Triggers pain animations and sounds in response to health loss
- **Math utilities** (`AngleVectors`, `DotProduct`, `VectorSubtract`, `VectorLength`): Computes damage direction and view-kick angles
- **Globals** (`cg`, `cgs`, `cg_entities[]`): Reads server snapshot state; writes frame-local UI and feedback state

## Design Patterns & Rationale

### 1. **Event Duality & Prediction Reconciliation**
The file distinguishes between **predictable events** (fired locally during movement prediction, then reconciled with server) and **external events** (server-pushed, always authoritative). This dual-path design allows:
- **Client-side responsiveness:** Local prediction fires events immediately (footsteps, weapon fire feedback).
- **Server correction:** If server disagrees on an event, `CG_CheckChangedPredictableEvents` re-fires the corrected version and logs a warning (`cg_showmiss`).

This is visible in the split between `CG_CheckPlayerstateEvents` (snapshot path) and `CG_CheckChangedPredictableEvents` (post-prediction path).

### 2. **Ammo Weighting Heuristic**
The `CG_CheckAmmo` function uses a two-tier weighting scheme:
- **Slow weapons** (rocket, grenade, rail, shotgun): 1000 ms per round
- **Fast weapons** (machinegun, etc.): 200 ms per round
- **Threshold:** 5000 ms total → flag as "low ammo"

This encodes game design intent: slow weapons are scarcer and more impactful, so firing one slow weapon feels more ammunition-depleting than firing many fast bullets. The metric is not "rounds remaining" but "estimated seconds until empty."

### 3. **Damage Feedback Geometry**
`CG_DamageFeedback` converts server-encoded yaw/pitch bytes into two outputs:
- **View-kick**: Roll and pitch applied to camera (kinesthetic feedback)
- **HUD indicator**: Screen-space X/Y position for a "you were hit from here" indicator

The math projects damage direction onto the camera's local axes, enabling screen-edge indicators to point toward the attacker. This is a classic game-feel technique: hurt direction + view kick → immersive combat feedback.

### 4. **Warning State as Bitfield**
Timelimit and fraglimit warnings use single-bit flags (1, 2, 4) to track which thresholds have been announced:
```
5 min warning → set bit 1
2 min (or 2-frag) warning → set bits 1|2
1 min/frag warning → set bits 1|2|4
```
This ensures each threshold triggers sound exactly once, and avoids spammy repeated warnings.

### 5. **Circular Buffer Indexing**
Predictable events and playerstate events use `i & (MAX_PS_EVENTS-1)` to index circular buffers. This is a classic optimization: assumes `MAX_PS_EVENTS` is a power of 2, avoiding modulo division.

## Data Flow Through This File

**Per-Frame Flow:**
1. **Snapshot arrives** → `cg_snapshot.c` calls `CG_TransitionPlayerState(current, previous)`
2. **Follow-mode check**: If `clientNum` changed, copy current into previous (clear stale state)
3. **Damage events**: If damageEvent incremented, call `CG_DamageFeedback` → mutate view-kick state
4. **Respawn**: If spawn count incremented, call `CG_Respawn` → reset prediction, open weapon UI
5. **Events loop**: Call `CG_CheckPlayerstateEvents` → fire entity events, log to predictable buffer
6. **Local sounds**: Call `CG_CheckLocalSounds` → detect stat deltas (hits, kills, rewards, timers) → queue sounds
7. **Ammo check**: Call `CG_CheckAmmo` → warn if low

**Data mutations:**
- `cg.damageX/Y/Value`, `cg.v_dmg_pitch/roll/time` (view feedback)
- `cg.lowAmmoWarning` (HUD state)
- `cg.timelimitWarnings`, `cg.fraglimitWarnings` (announcement dedup)
- `cg.weaponSelect`, `cg.weaponSelectTime` (UI state)
- `cg.thisFrameTeleport`, `cg.duckChange` (prediction hints)
- `cg.rewardStack`, `cg.rewardSound/Shader/Count[]` (HUD reward display)

All of these are read later in the same frame by draw/render code.

## Learning Notes

**Prediction Reconciliation Pattern:** This file demonstrates a sophisticated client-side prediction system. The game predicts locally (firing events immediately), but the server is authoritative. If prediction diverges, the event system re-fires the corrected version. Modern networked shooters use similar patterns (e.g., Overwatch, Valorant).

**Circular Event Buffers:** The use of power-of-2 circular buffers with bitwise indexing is idiomatic to early-2000s game engines (predates heap allocation per-frame). Modern engines might use dynamic arrays or pre-allocated pools.

**Sound Layering & Channels:** The distinction between `CHAN_LOCAL_SOUND` (player feedback, non-exclusive), `CHAN_ANNOUNCER` (announcer voice, exclusive), and buffered sounds shows how to prioritize multiple sound categories without audio clipping.

**Magic Numbers:** The file contains several magic constants—health threshold 40, ammo weights 1000/200, kick clamps 5–10—that encode game balance tuning. These would benefit from `#define` names (e.g., `LOW_HEALTH_THRESHOLD`, `AMMO_WEIGHT_SLOW`).

## Potential Issues

1. **Hardcoded ammo weights and thresholds** (lines 55–65, 76): If weapon balance changes, these values must be updated manually. No centralized "weapon database" drives them.

2. **Prediction reconciliation complexity** (lines 252–290): The logic for detecting changed predictable events is subtle. If `MAX_PS_EVENTS` is small, the circular buffer can wrap and mask real changes. Requires careful documentation.

3. **Damage kick saturated at 10** (line 108): The view-kick is clamped to [5, 10] before direction math, but again clamped at line 174 after direction math—redundant safety clamping suggests uncertainty about the bounds.

4. **Magic `DAMAGE_TIME` constant** (line 171): The duration of damage feedback is probably defined in a header, but tying it to this struct mutation makes it easy to desynchronize if changed elsewhere.

5. **No null checks on event pointers**: `cent->currentState` is assumed valid in `CG_CheckPlayerstateEvents`. If `cg_entities[]` is corrupted or `clientNum` is out of bounds, this could crash (though cgame should guarantee valid pointers).
