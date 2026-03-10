# code/game/g_active.c — Enhanced Analysis

## Architectural Role

`g_active.c` is the per-frame execution core of the **Game VM** subsystem, sitting at the intersection of the Server layer (which delivers `usercmd_t` commands) and the shared physics layer (`bg_pmove.c`). It is one of two files the **server** calls directly into the game VM each frame: `ClientThink` for command-driven updates, and `ClientEndFrame` (called from `g_main.c:G_RunFrame`) for end-of-frame flush. The file is the authoritative enforcer of server-side state — it owns the `playerState_t → entityState_t` synchronization that feeds the server's snapshot-building pipeline in `sv_snapshot.c`, and it is the sole place where environment hazards, powerup timers, and inactivity enforcement are applied before a snapshot is sent.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_client.c`** — calls `ClientThink(clientNum)` via the game VM's `vmMain(GAME_CLIENT_THINK, ...)` dispatch whenever a new `usercmd_t` arrives from the network. The server never sees the internals; it only holds the `clientNum`.
- **`code/game/g_main.c:G_RunFrame`** — calls `G_RunClient(ent)` for every bot or synchronous-mode client once per server tick, and calls `ClientEndFrame(ent)` for every active client at the tail of every frame.
- **`code/game/ai_main.c`** — `BotTestAAS` is invoked inside `ClientThink_real` to validate bot AAS navigation state; this is the one place where the game VM actively calls back into the bot AI stack during player simulation.
- **`code/game/g_combat.c`** — writes `client->damage_blood`, `damage_armor`, `damage_knockback`, `damage_from`, `damage_fromWorld` across the frame; `P_DamageFeedback` is the deferred consumer that reads and clears these fields.
- **`code/game/g_weapon.c`** — `CheckGauntletAttack` and `Weapon_HookFree` are called inside `ClientThink_real`; `FireWeapon` is invoked from `ClientEvents` when `EV_FIRE_WEAPON` is processed server-side.

### Outgoing (what this file depends on)

- **`bg_pmove.c` (shared physics)** — `Pmove` is the central call; its output `pm.ps` is the authoritative `playerState_t` that drives all subsequent state synchronization.
- **`bg_misc.c`** — `BG_PlayerStateToEntityState`/`BG_PlayerStateToEntityStateExtraPolate` translate the physics-computed player state into the network-transmittable `entityState_t`; `BG_PlayerTouchesItem` provides trigger overlap for item pickup without requiring actual bounding-box contact.
- **`g_combat.c`** — `G_Damage`, `G_RadiusDamage` (via fall/water damage paths), `G_AddEvent`, `G_Sound`, `G_TempEntity`, `G_SoundIndex`.
- **`g_weapon.c`** — `FireWeapon`, `CheckGauntletAttack`, `Weapon_HookFree`.
- **`g_client.c` / `g_misc.c`** — `respawn`, `TeleportPlayer`, `SelectSpawnPoint`, `Drop_Item`.
- **`trap_*` syscalls (g_syscalls.c → server)** — `trap_Trace`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_EntitiesInBox`, `trap_EntityContact`, `trap_GetUsercmd`, `trap_DropClient`, `trap_SendServerCommand`, `trap_PointContents`.
- **Global `level`** — reads `level.time`, `level.clients`, `level.snd_fry`, `level.intermissiontime`, `level.intermissionQueued`; the file has no file-static state of its own.

## Design Patterns & Rationale

**Dual entry-point pattern (`ClientThink` / `G_RunClient`)**: The engine can call `ClientThink` multiple times per server frame for fast clients (those that submit commands faster than the tick rate). `G_RunClient` forces `serverTime = level.time` to handle bots and the synchronous flag. This gives bots and players unified simulation while allowing variable-rate human input.

**Deferred damage aggregation**: Rather than encoding pain feedback on every `G_Damage` call, damage is accumulated in `damage_blood` / `damage_armor` / `damage_from` throughout the frame and flushed in one atomic write inside `P_DamageFeedback`. This collapses multiple hits in one frame into a single directional blend and a single debounced pain sound, preventing visual/audio spam.

**Shared physics compilation (`bg_pmove.c`)**: The identical `Pmove` source is compiled into both the game VM (authoritative) and cgame VM (predictive). `g_active.c` is the server-side caller; `cg_predict.c` is the client-side caller. The tradeoff is a rigidly constrained shared-code surface — any physics change must remain deterministic across both compilation units.

**Predictable event forwarding (`SendPendingPredictableEvents`)**: Events that cgame predicts (fired weapons, etc.) are re-sent via a temp entity so *other* clients who cannot predict them still receive authoritative notification. This is a consequence of having prediction without a full rollback/reconciliation system.

## Data Flow Through This File

```
usercmd_t (from network or bot EA layer)
    → ClientThink_real: command time validation, anti-cheat clamping
    → Pmove(): physics simulation
    → playerState_t (authoritative position, velocity, flags)
    → BG_PlayerStateToEntityStateExtraPolate(): → entityState_t (snapshot-ready)
    → ClientEvents(): process new ps.events[] server-side
          → FireWeapon / G_Damage / TeleportPlayer / respawn
    → G_TouchTriggers(): broad+narrow-phase trigger overlap → touch callbacks
    → ClientImpacts(): Pmove touch list → entity touch callbacks
    → ClientTimerActions(): once-per-second health/armor/ammo mutations
    → SendPendingPredictableEvents(): temp entity for other clients

    [end of frame, via ClientEndFrame]
    → P_WorldEffects(): env damage accumulation → G_Damage
    → P_DamageFeedback(): flush damage_* fields → ps.damagePitch/Yaw/Count
    → G_SetClientSound(): update ps.loopSound
    → BG_PlayerStateToEntityStateExtraPolate(): final snapshot sync
    → SendPendingPredictableEvents(): any remaining predictable events
```

Key state transitions: `STAT_HEALTH` decrement (world effects) → `EV_PAIN` event → `ps.damageCount` (feedback for cgame pain blend) → snapshot delivery.

## Learning Notes

- **The `bg_*` prediction contract** is the most instructive pattern here: physics runs identically on client and server by sharing source, not by network synchronization. Modern engines (Unreal's CharacterMovementComponent, Valve's prediction system) follow the same principle but with more sophisticated rollback. The Q3 version is intentionally minimal — there is no server-side rollback; divergence manifests as visible snapping.
- **Command time anti-cheat** (`level.time ± 200ms`, max 1000ms gap) is simple rate-limiting, not cryptographic. It prevents speedhacks but not lag-exploitation cheats. Modern engines add sequence number validation and client-specific timing windows.
- **`timeResidual` accumulator** in `ClientTimerActions` is a classic fixed-timestep-within-variable-timestep pattern: accumulate wall-clock `msec`, drain in 1000ms increments. This decouples once-per-second logic from the actual frame rate without scheduling overhead.
- **Spectator/follow separation**: `SpectatorThink` runs a full pmove for free-flying spectators but skips it entirely for follow-spectators (who just copy the followed player's `playerState_t` in `SpectatorClientEndFrame`). This is an early example of "spectator as view mode, not physics object."
- **No scene graph / ECS**: Each entity has a monomorphic `think` function pointer — Q3's entity system is a flat array of tagged unions driven by function pointers, the pattern that ECS was explicitly designed to replace.

## Potential Issues

- **`rand()&1` in `P_WorldEffects`** for gurp sound selection is non-deterministic and not seeded from any game state. This is safe (it's cosmetic audio) but means cgame cannot predict which sound will play — a minor but inconsistent exception to the prediction model.
- **`damage_knockback` is accumulated but never read** in `P_DamageFeedback` — it is cleared but its value is not encoded into `playerState_t`. The knockback is applied directly in `g_combat.c` via velocity mutation, making the accumulator vestigial.
- **MISSIONPACK ammo regen** iterates an 11-element stack-allocated `weapList[]` every frame for every client holding `PW_AMMOREGEN`, which is a minor hot-path allocation that could be a static const array.
