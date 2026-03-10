# code/game/ai_team.c — Enhanced Analysis

## Architectural Role

This file implements the **team coordination layer** of the bot AI system—the strategic leadership subsystem that translates game objectives and team composition into tactical role assignments. It sits at the boundary between individual bot decision-making (`ai_dmnet.c`/`ai_dmq3.c`, which execute per-bot FSMs) and the game mode rules (CTF, 1FCTF, Obelisk, Harvester). A single designated team leader bot periodically recalculates teammate proximity to home objectives and broadcasts role orders (defend/attack/escort) to the squad, enabling emergent multi-bot tactics.

## Key Cross-References

### Incoming (who depends on this file)

- **`ai_main.c` / bot frame loop**: Calls `BotTeamAI()` once per bot per frame via the game's primary bot think hook (`BotAIStartFrame`). Only the leader actually executes; others skip early.
- **Individual bot FSMs (`ai_dmnet.c`)**: Bots read and respect role assignments stored in `ctftaskpreferences[]` via `BotGetTeamMateTaskPreference()` to bias their goal selection and movement decisions.
- **Game mode orchestrators (`ai_dmq3.c`)**: Consume the global `ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk` goal structures set by this file's sorting functions.

### Outgoing (what this file depends on)

- **botlib navigation**: `BotPointAreaNum()`, `trap_AAS_AreaTravelTimeToGoalArea()` for proximity calculations; `BotAI_GetClientState()` to read client origin.
- **Game state queries**: `trap_GetConfigstring()` for player roster; `BotSameTeam()`, `BotTeam()`, `ClientName()`, `ClientFromName()` for player/team lookups.
- **Chat/voice subsystem**: `BotAI_BotInitialChat()`, `trap_BotGetChatMessage()`, `trap_BotEnterChat()`, `trap_BotQueueConsoleMessage()`, `trap_EA_Command()` to deliver orders. Voice delivery is conditional on `MISSIONPACK` compile flag.
- **Global game state**: Reads `gametype`, `g_entities[]`, `notleader[]` array (notleader bots flagged by human players).

## Design Patterns & Rationale

| Pattern | Where | Why |
|---------|-------|-----|
| **Centralized team leader** | `BotValidTeamLeader()`, `FindHumanTeamLeader()`, `BotTeamAI()` gating | Simplifies coordination: one bot issues orders every ~250ms instead of n bots debating. Prefers humans to preserve player agency. |
| **Insertion sort by travel time** | `BotSortTeamMatesByBaseTravelTime()` | O(n²) cost is acceptable (n ≤ 64 clients); produces stable ordering needed for role assignment heuristics. |
| **Persistent task preference cache** | `ctftaskpreferences[]`, `BotSetTeamMateTaskPreference()` | Allows human players to set bot roles via chat; name-field validation mitigates stale slot reuse. Survives across map changes. |
| **Lazy strategy toggle** | `BotTeamAI()` random toggle if `!lastscorechange` | Avoids expensive recalculation every frame; 4-minute inactivity triggers passive→aggressive flip with 40% probability. |
| **Conditional compilation for voice** | `#ifdef MISSIONPACK` in chat functions | Base Q3A uses text-only; MissionPack adds voice commands. Avoids runtime feature detection overhead. |
| **Re-dispatch on flag-status change** | `BotTeamAI()` event-driven execution | Expensive AAS queries only when objective state changes (flag moved, captured), not every frame. |

## Data Flow Through This File

```
BotTeamAI() per frame
  ├─ Validate/elect leader (query g_entities[], configstrings)
  │
  └─ [If leader] Evaluate trigger conditions:
       ├─ Team size change → recalculate
       ├─ Flag status changed → recalculate
       └─ Timeout (>250ms) → recalculate
       
       For each trigger:
         ├─ BotSortTeamMatesByBaseTravelTime()
         │  └─ Consult AAS: travel times to home flag/obelisk
         │
         ├─ BotSortTeamMatesByTaskPreference()
         │  └─ Re-order: defenders first, then roamers, then attackers
         │
         └─ Mode dispatcher (BotCTFOrders, etc.)
            ├─ Encode flag status (binary: red home? blue home?)
            ├─ Branch on team size (case 2,3,default)
            └─ For each teammate:
               ├─ BotAI_BotInitialChat() queue message
               ├─ BotSayTeamOrder() dispatch (text or voice)
               └─ Update ctftaskpreferences[] with assigned role
```

**Data residency**: Leader election and strategy state live in `bs` (individual bot state). Task preferences live in global `ctftaskpreferences[]` indexed by client slot. AAS proximity queries are read-only. Chat messages are queued in botlib's chat state.

## Learning Notes

### Idiomatic to This Era / Engine

1. **No ECS**: All state is imperative + global arrays. Task preferences are stored as a flat array indexed by client slot, not as entity-component relations.
2. **Travel-time heuristics**: Proximity to home base is the sole spatial metric for role assignment. No pathfinding cost, no avoidance of enemy-held territory—simpler but cruder.
3. **Synchronous ordering**: All orders issued within the same frame via blocking chat queue operations. No asynchronous coordination or consensus.
4. **Manual preference persistence**: The game doesn't provide a built-in player-attribute system; preferences are hand-rolled and validated by re-reading player names from configstrings.
5. **Compile-time feature gating**: Voice chat is `#ifdef MISSIONPACK`, not a runtime feature flag. Reflects era where console/expansion packs bundled feature variants.
6. **Conditional text vs voice**: Base game uses text chat; MissionPack uses voice-only. No fallback or coexistence.

### Modern Engines Do Differently

- **Behavior trees or hierarchical FSMs**: Coordinate multi-agent behavior via explicit task graphs, not hard-coded role logic per game mode.
- **Perception/target sharing**: Bots would communicate enemy sightings, not just role assignments.
- **Spatial partitioning**: Modern engines use influence maps or heat maps to dynamically assign roles based on threat distribution, not just proximity.
- **Scripted AI subsystems**: Role logic would live in data-driven configs, not C code branching on game mode.

### Connections to Game Engine Concepts

- **Hierarchical bot AI**: This file implements the *squad-level* layer (team coordination) above *individual-level* (FSM in `ai_dmnet.c`) and below *global-level* (game rules). Classic 3-tier AI hierarchy.
- **Spatial reasoning**: Heavy use of AAS (annotated area system) for understanding map topology. This is Q3A's "semantic spatial layer" above BSP geometry.
- **Command dispatch pattern**: Orders are issued via a central chat system that enqueues text/voice commands—analogous to a command buffer in a graphics engine, but for AI communication.

## Potential Issues

| Issue | Severity | Location | Notes |
|-------|----------|----------|-------|
| **Buffer overflow in `strcpy`** | High | `BotSetTeamMateTaskPreference:173` | `strcpy(ctftaskpreferences[teammate].name, ...)` is unbounded. Should use `Q_strncpyz()` or similar. |
| **Uninitialized global state** | Medium | `ctftaskpreferences` global | Array is never explicitly zeroed on game load. Relies on BSS zero-initialization (fine in practice, but not defensive). |
| **Leader validation insufficient** | Low | `BotValidTeamLeader:77` | Checks name mapping, not whether leader is alive, on correct team, or not in `notleader[]`. Could order a dead player. |
| **Stale preference detection race** | Low | `BotGetTeamMateTaskPreference:211` | Name comparison uses `Q_stricmp()` against configstring-derived name; if a slot is reused mid-frame, preference might briefly apply to the wrong player. Unlikely in practice due to frame-boundary safety. |
| **Strategy not synchronized across bots** | Low | `BotTeamAI:2000+` | Only leader bot's `bs->ctfstrategy` is toggled; other bots' copies may be stale if they were elected earlier. Not a correctness bug (only affects leader logic), but inconsistent state. |
| **Hard-coded voice command strings** | Medium | `BotVoiceChat` / `BotSayVoiceTeamOrder` | Voice chat names (`VOICECHAT_GETFLAG`, etc.) are literal strings passed to `trap_EA_Command`. No validation that voice packs include the command. |
