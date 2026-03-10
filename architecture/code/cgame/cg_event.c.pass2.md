# code/cgame/cg_event.c — Enhanced Analysis

## Architectural Role

`cg_event.c` serves as the event translation bridge between cgame's snapshot consumption layer and the wider effect/sound/rendering subsystems. It sits at a critical juncture: server snapshots arrive with `EV_*` event codes in `entityState_t`; cgame's job is to immediately materialize these abstract events into observable client effects (audio, visuals, HUD feedback). This is the sole demultiplexer for all server-side events reaching the local player, making it load-bearing for perceived game responsiveness.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_ents.c:CG_AddPacketEntities()`** — calls `CG_CheckEvents()` once per visible packet entity per frame; event deduplication happens here before dispatch
- **`cg_playerstate.c:CG_CheckPlayerstateEvents()`** — routes predicted/server player events (footsteps during prediction, pain) through `CG_EntityEvent()`
- **`cg_draw.c`** — consumes `CG_PlaceString()` for rank formatting in scoreboard/postgame screens
- **Obituary display** — triggered whenever a `EV_OBITUARY` event arrives; writes to `cg.killerName` for HUD state

### Outgoing (what this file depends on)

- **Effect subsystems** (`cg_effects.c`): `CG_SmokePuff`, `CG_SpawnEffect`, `CG_MissileHitPlayer`, `CG_MissileHitWall`, `CG_RailTrail`, `CG_Bullet`, `CG_ShotgunFire`, `CG_GibPlayer`
- **Weapon subsystems** (`cg_weapons.c`): `CG_FireWeapon`, `CG_OutOfAmmoChange`; also references `bg_itemlist[]` (shared deterministic game data)
- **Sound system**: `trap_S_StartSound`, `trap_S_StopLoopingSound`, `CG_AddBufferedSound`, `CG_CustomSound` for gender-specific pain/item audio
- **HUD/Draw**: `CG_CenterPrint` (frag notifications, item use), `CG_ScorePlum` (damage indicators)
- **Network/State**: reads `cg.snap->ps` (current player state) for frag messages and rank computation; mutates `cg` globals (`landTime`, `powerupActive`, etc.)
- **Shared game constants**: `MOD_*` (damage mode) and `EV_*` (event code) enums from `q_shared.h` ensure consistency with server

## Design Patterns & Rationale

**Central event dispatch via switch**: Large `switch(event)` statement is straightforward but scales poorly; this era predates data-driven event tables. The structure makes adding new `EV_*` types trivial — just add a case.

**Event deduplication via `previousEvent`**: Each `centity_t` tracks its last-seen event code; `CG_CheckEvents()` fires only on transition (`!= previousEvent`). Simple, O(1), and prevents re-triggering on unchanged snapshots — critical for avoiding sound spam and duplicate effects.

**Gender-aware obituary text**: `ci->gender` is consulted to select pronoun, reflecting late-90s multiplayer culture and localization awareness. Message templates (e.g., "tripped on his/her/its own grenade") are baked into the switch rather than table-driven.

**Rate-limiting by entity**: `CG_PainEvent()` uses per-entity `ce->pe.painTime` to throttle to ~2 sounds/second, preventing audio feedback loops from rapid damage.

**MOD-indexed message selection**: Obituary messages are selected by MOD (method of death) rather than weapon; this decouples kill messages from weapon state and allows splash-damage variants (e.g., `MOD_ROCKET` vs. `MOD_ROCKET_SPLASH`) to have distinct phrasing.

## Data Flow Through This File

1. **Server→Network**: Server entity event emitted in `entityState_t.event` field (encodes MOD/weapon/damage info in `eventParm`)
2. **Network→cgame snapshot**: `CL_ParseSnapshot()` delivers snapshot to cgame; events embedded in entity deltas
3. **cgame tick**: `CG_AddPacketEntities()` iterates visible entities, calls `CG_CheckEvents()`
4. **Deduplication**: `CG_CheckEvents()` compares `currentState.event != previousEvent`; if new, computes position via `BG_EvaluateTrajectory()` and forwards to `CG_EntityEvent()`
5. **Dispatch & execution**: `CG_EntityEvent()` reads event code and routes:
   - **Impact events** → effect spawning (`CG_MissileHitWall`, `CG_RailTrail`)
   - **Movement events** → sound + HUD feedback (`CG_PainEvent`, `CG_ItemPickup`)
   - **Death events** → obituary + scoring (`CG_Obituary` mutates `cg.killerName`)
   - **Audio** → trap sound system with positional context
6. **Side effects**: Updates `cg.itemPickup`, `cg.powerupActive`, entity pain direction; may trigger weapon select or center-print

## Learning Notes

**Snapshot event coupling**: Unlike modern ECS engines, events are tightly bound to snapshot entity deltas. This forces deduplication logic into cgame rather than delegating to an event queue — a tradeoff for tight coupling but simple ownership semantics.

**MOD_* as cross-subsystem vocabulary**: The `MOD_*` enum family (defined in `q_shared.c`) flows through game VM, network serialization, and cgame without translation. This is idiomatic Q3A: shared constants act as a minimal contract between subsystems.

**Gender in obituaries**: Quake III's "gender" system (GENDER_MALE/FEMALE/NEUTER from `bg_public.h`) shows mid-90s multiplayer design priorities. Modern engines would template strings or use localization tables; here, gender selection is hardcoded inline.

**No event ordering guarantee**: Because events dedup on per-entity basis, rapid events (e.g., triple footsteps in one frame) may coalesce or reorder across the network. The design assumes events are idempotent or rare enough that loss is acceptable.

## Potential Issues

- **Rate-limiting edge case**: `CG_PainEvent()` throttle uses `cg.time - ce->pe.painTime < 500` but `cg.time` can jump on lag spikes, potentially skipping pain feedback entirely.
- **No bounds checking on derived indices**: `itemNum = (es->event & ~EV_EVENT_BITS) - EV_USE_ITEM0` assumes `EV_USE_ITEM0` is carefully defined; off-by-one errors would silently index wrong item.
- **Static string buffer reuse**: `CG_PlaceString()` returns pointer to static `str[64]`, making it unsafe if caller doesn't sprintf/copy immediately. Scoreboard drawing must be careful not to nest calls.
- **MissionPack fragmentation**: `#ifdef MISSIONPACK` gates are scattered throughout switch cases, making it hard to diff base-Q3A and TA event handling at a glance.
