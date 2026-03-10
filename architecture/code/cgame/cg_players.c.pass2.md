# code/cgame/cg_players.c — Enhanced Analysis

## Architectural Role

This file is the **exclusive client-side renderer of player entities** and the primary consumer of server-delivered snapshot player state. It bridges the server's authoritative `entityState_t`/`playerState_t` data (velocity, animation indices, team, effects) with the renderer and sound subsystems by assembling three-part skeletal models, evaluating animation frames, computing per-frame orientation, and attaching dynamic effects (powerups, flags, weapon trails). It also owns the client-info media-loading pipeline, which is decoupled from snapshot consumption via deferred loading to prevent frame stutters during gameplay.

## Key Cross-References

### Incoming (who depends on this file)
- **`cg_ents.c:CG_AddPacketEntities`** — Calls `CG_Player` for every `ET_PLAYER` entity each frame; this is the primary render entry point
- **`cg_servercmds.c:CG_SetConfigstring`** — Triggers `CG_NewClientInfo` when a player's configstring changes (join/rejoin/rename/model change)
- **`cg_main.c:CG_DrawActiveFrame`** — Calls `CG_LoadDeferredPlayers` each frame during scoreboard display to drain the deferred media-load queue
- **`cg_weapons.c`, `cg_effects.c`, `cg_localents.c`** — Call `CG_CustomSound` to resolve wildcard sound names for player footsteps, pain cries, and taunt audio
- **`cg_predict.c:CG_BuildSolidList`** — May reference `cgs.clientinfo[i].bbox` for collision prediction

### Outgoing (what this file depends on)
- **Renderer (`trap_R_*`)** — `RegisterModel`, `RegisterSkin`, `RegisterShaderNoMip`, `AddRefEntityToScene`, `LerpTag`, `DrawModel`; the file submits the majority of visible refEntities each frame
- **Sound (`trap_S_*`)** — `RegisterSound`, `StartSound`, `StartLocalSound` via `CG_CustomSound` resolution
- **Filesystem (`trap_FS_*`)** — Reads `animation.cfg` during media load; uses file-exists probes to implement model/skin fallback chains
- **Shared game code** — `BG_EvaluateTrajectory` (trajectory.c equivalent) for player origin interpolation
- **Other cgame modules** — `CG_PositionRotatedEntityOnTag`, `CG_AddPlayerWeapon`, `CG_AddRefEntityWithPowerups`, `CG_PlayerPowerups`, `CG_PlayerSprites`, `CG_PlayerShadow`, `CG_PlayerSplash`, `CG_PlayerTokens` (MISSIONPACK), `CG_BreathPuffs`, `CG_DustTrail`, etc.
- **Global state** — `cgs.clientinfo[]` (cached per-client media), `cg_entities[]` (per-entity anim/lerp state), `cgs.gametype`, `cgs.protocol` (for new-anim detection)

## Design Patterns & Rationale

### 1. **Deferred Media Loading**
`CG_NewClientInfo` → `CG_SetDeferredClientInfo` queues the load; `CG_LoadDeferredPlayers` drains it **only during scoreboard display**. This avoids model-registration stalls during active gameplay. The tradeoff: first-time player renders may use a fallback/default model until the next frame's deferred batch.

### 2. **Hierarchical Body Animation with Yaw/Pitch Swing**
Legs, torso, and head are animated independently but coupled via `CG_PlayerAngles`, which computes **yaw swing** (head tracking input), **pitch fraction** (weapon aiming), and **pain twitch** offsets applied post-lerp. This decoupling allows shared animations (e.g., `TORSO_STANCE` looping while `LEGS_RUN` advances).

### 3. **Animation Frame Lerping with Backlerp**
`CG_RunLerpFrame` advances frame indices each tick and computes `backlerp` (0–1000) to blend between frame N and N+1. The animation system is **frame-based, not skeleton-based**, with no IK or procedural blending—all transitions are hand-authored in `animation.cfg`.

### 4. **Defensive Model/Skin Resolution with Fallback Chains**
`CG_FindClientModelFile` and `CG_FindClientHeadFile` implement multi-level fallbacks:
- Try `models/players/characters/<model>/<skin>_<team>.<ext>`
- Try `models/players/<model>/<skin>_<team>.<ext>` (no "characters" subfolder)
- Try without skin specifier, without team specifier, etc.

This suggests **asset packs shipped incomplete** or **different platforms had different asset layouts**.

### 5. **Team/Gametype Awareness at Render Time**
The file hardcodes logic to choose `team = "red"/"blue"/"default"` based on `ci->team` and `cgs.gametype >= GT_TEAM`. This couples asset resolution to game rules, making it fragile to future game modes.

### 6. **New-Animation Detection via Tag Query**
`CG_LoadClientInfo` queries `tag_flag` via `trap_R_LerpTag` to detect if a model supports **Team Arena (MISSIONPACK) animation system** with embedded flag attachment. This is a runtime feature-detection hack that avoids version checks.

### 7. **Sound Registration Batching**
All custom sounds are registered once in `CG_LoadClientInfo` and cached in `ci->sounds[]`. `CG_CustomSound` is a O(1) array lookup, not a per-call registration.

## Data Flow Through This File

```
INITIALIZATION PHASE (server → client)
───────────────────────────────────────
configstring change (player join/change)
    ↓
CG_NewClientInfo(clientNum)
    ├─ Parse player attributes (name, color, model, head, team)
    ├─ Check for existing loaded model (scan clientinfo cache for reuse)
    ├─ [If memory low] Defer load via CG_SetDeferredClientInfo → return
    └─ [Else] CG_LoadClientInfo(ci)
        ├─ CG_RegisterClientModelname(ci, ...) 
        │  ├─ trap_R_RegisterModel() ×3 (legs, torso, head)
        │  ├─ CG_ParseAnimationFile(animation.cfg) → fill ci->animations[]
        │  ├─ CG_RegisterClientSkin(ci, ...) → fill ci->legsSkin, torsoSkin, headSkin
        │  └─ trap_R_LerpTag(tag_flag) → detect newAnims flag
        ├─ trap_S_RegisterSound() ×MAX_CUSTOM_SOUNDS → fill ci->sounds[]
        └─ CG_ResetPlayerEntity() ×N → clear all entities using this client


DEFERRED LOADING PHASE (every frame during scoreboard)
──────────────────────────────────────────────────────
CG_LoadDeferredPlayers()
    └─ For each deferred clientinfo: CG_LoadClientInfo(ci) [if memory OK]


PER-FRAME RENDERING PHASE (snapshot → 3D scene)
────────────────────────────────────────────────
CG_AddPacketEntities()
    └─ [For each ET_PLAYER in snapshot]
        └─ CG_Player(cent)
           ├─ CG_PlayerAnimation(cent) 
           │  ├─ Evaluate legs anim:  CG_RunLerpFrame(ci, &cent->pe.legs, ...)
           │  ├─ Evaluate torso anim: CG_RunLerpFrame(ci, &cent->pe.torso, ...)
           │  └─ Evaluate flag anim:  CG_RunLerpFrame(ci, &cent->pe.flag, ...)
           │
           ├─ CG_PlayerAngles(cent, legs_axis, torso_axis, head_axis)
           │  ├─ CG_SwingAngles() → compute yaw/pitch swing offsets
           │  ├─ CG_AddPainTwitch() → apply damage feedback
           │  └─ AnglesToAxis() ×3 → convert to rotation matrices
           │
           ├─ Assemble & submit refEntities:
           │  ├─ legs refEntity (model, skin, animation frame, axis)
           │  ├─ torso refEntity (attached to legs via tag_torso)
           │  ├─ head refEntity (attached to torso via tag_head)
           │  ├─ [If flag carrying] flag refEntity + animation
           │  ├─ CG_AddPlayerWeapon(cent, ...)
           │  └─ [If powerups] CG_AddRefEntityWithPowerups() overlay passes
           │
           ├─ Attach visual effects:
           │  ├─ CG_PlayerPowerups() → quad/regen/battlesuit/invis overlays
           │  ├─ CG_PlayerSprites() → floating name/damage sprites
           │  ├─ CG_PlayerShadow() → mark decal beneath feet
           │  ├─ CG_PlayerSplash() → water entry/exit splash
           │  └─ [MISSIONPACK] CG_BreathPuffs(), CG_DustTrail(), etc.
           │
           └─ [If footstep sound event] Play via CG_CustomSound → ci->sounds[]


SOUND PLAYBACK PATH
───────────────────
CG_Event() [from entity event]
    └─ CG_CustomSound(clientNum, "*death1.wav", ...)
        ├─ Lookup in cg_customSoundNames[] → index i
        ├─ Return cgs.clientinfo[clientNum].sounds[i]
        └─ [Or trap_S_RegisterSound(soundName) for non-wildcard names]
            └─ Play via trap_S_StartSound(...)
```

## Learning Notes

### 1. **Frame-Based vs. Skeleton-Based Animation**
This file uses a **completely frame-based animation system**:
- Each animation has `firstFrame`, `numFrames`, `loopFrames`, `frameLerp` (ms per frame)
- No bones, IK, or procedural blending; all blend data is pre-authored
- Compare to modern engines (Unreal, Godot): they use skeletal animation with runtime blending and procedural overlays

This is idiomatic to **Quake III's offline toolchain** (Max/Maya models exported as MD3 keyframe data).

### 2. **Snapshot-Driven Rendering vs. Entity Simulation**
Unlike modern engines with persistent entity objects updated by physics/logic systems, Q3A is **snapshot-driven**: the client has no entity "state machine," only a per-frame refEntity assembly step. This is efficient for LAN play (low latency) but complicates client-side prediction (which happens separately in `cg_predict.c`).

### 3. **Team Color Hardcoding**
The file assumes exactly two teams (`TEAM_RED` vs. `TEAM_BLUE`) and bakes this into asset-finding logic. Mods with custom team counts or colors would require code changes here.

### 4. **Animation.cfg as Data-Driven Design**
Instead of hardcoding animation indices, `CG_ParseAnimationFile` reads a text config per model. This was a way to **ship new player models without code recompilation**. Modern engines use asset pipelines; Q3A's approach is simpler but inflexible (e.g., no variant support).

### 5. **Fallback Chains as Robustness**
The multiple path attempts for model/skin loading suggest the authors encountered **incomplete or missing asset packs** during development. This defensive pattern is rare in modern engines but common in shipped Q3A mods.

### 6. **Deferral as Performance Optimization**
`CG_LoadDeferredPlayers` shows the developers knew model registration was expensive and engineered a heuristic: defer loads until the scoreboard is visible (latency-insensitive moment). This is a precursor to modern **frame-budgeting** techniques.

## Potential Issues

1. **Animation Table Overflow**: Missing Team Arena animations are aliased to `TORSO_GESTURE`, but only if the animation config doesn't supply them. If a legacy model lacks `TORSO_GESTURE`, aliasing will reference uninitialized data.

2. **Team/Gametype Coupling**: Asset selection is hardcoded per team and gametype. Custom game modes cannot introduce new team colors or asset naming conventions without code changes.

3. **No Animation Blending Between Overlays**: Powerup/flag states switch instantly; there's no transition animation. A player grabbing a flag will snap to the flag-carry pose rather than blending.

4. **Memory Pressure During Deferred Load**: If too many clients join simultaneously, the deferred queue grows unchecked. If memory remains low, clients will render with wrong models until the queue drains (potentially many frames).

5. **Hardcoded "characters/" Subfolder Logic**: The fallback from `models/players/<model>` to `models/players/characters/<model>` is a special case. Custom asset layouts cannot be expressed without code changes.

6. **Animation Frame Granularity**: Frame-based lerping (not bone-based) means animation quality depends on capture frame rate. 30 fps animations recorded at 15 fps will appear choppy on a 60 fps client if `frameLerp` values aren't tuned.
