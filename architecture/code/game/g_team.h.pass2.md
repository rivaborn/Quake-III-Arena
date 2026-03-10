# code/game/g_team.h — Enhanced Analysis

## Architectural Role

This header file is the **team/CTF game rules interface** within the Game VM subsystem. It bridges server-side game logic across multiple files (combat, item pickup, spawning, player lifecycle) by centralizing all CTF-specific scoring constants and declaring the public team-management function API. The conditional `#ifdef MISSIONPACK` block demonstrates Quake III's multi-variant build strategy: the same source tree produces both base-Q3A (simpler CTF rules) and Team Arena (richer bonus system) by recompiling with different balance defines.

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/g_combat.c**: Calls `Team_FragBonuses()` and `Team_CheckHurtCarrier()` during damage application to award proximity/carrier-protection bonuses
- **code/game/g_items.c**: Calls `Pickup_Team()` when a player collides with a flag entity; drives flag-carrier state
- **code/game/g_spawn.c**: Calls `SelectCTFSpawnPoint()` to assign spawn locations based on team and game state
- **code/game/g_client.c**: Calls `TeamplayInfoMessage()` and `Team_GetLocation()` for HUD team status updates
- **code/game/g_main.c**: Calls `Team_InitGame()` at level load to initialize flag and team state
- **code/game/g_missile.c / other**: May use utility functions like `OtherTeam()`, `TeamName()`, `TeamColorString()`

### Outgoing (what this calls)
- **code/game/g_local.h / q_shared.h**: Type imports (`gentity_t`, `team_t`, `vec3_t`, `qboolean`, `int`, `const char*`)
- **code/game/g_team.c**: Implementation of all declared functions; uses the `CTF_*` constants to control scoring and entity behavior
- Implicitly: **code/qcommon** (via syscalls in implementation): `trap_Trace`, `trap_LinkEntity`, `trap_SetConfigstring`, `trap_SendServerCommand` for world effects

## Design Patterns & Rationale

1. **Compile-Time Game Variant Selection**: The `#ifdef MISSIONPACK` pattern allows two entirely different scoring systems (CTF bonuses 5→100 for base; 1→5 for base; up to 20 for Team Arena) to coexist in source but diverge at build time. This was necessary in the 2000s when binary size and runtime memory mattered; avoids runtime branching overhead and simplifies the code flow in `Team_FragBonuses`.

2. **Constants-First Interface Design**: Unlike typical headers (which export functions and types), this file is ~80% constants. This reflects **game-balance-as-configuration**: all tunable parameters are visible and editable without touching implementation. Changing `CTF_CAPTURE_BONUS` from 100 to 120 requires only a recompile, not code logic changes.

3. **Radius + Timeout Pairs**: `CTF_TARGET_PROTECT_RADIUS` / `CTF_ATTACKER_PROTECT_RADIUS` and corresponding `TIMEOUT` constants form coherent tuning pairs — spatial and temporal thresholds for bonus eligibility. This is a disciplined way to parameterize proximity-based game rules.

4. **Utility Function Exports**: Functions like `OtherTeam()`, `TeamName()`, `TeamColorString()` are thin wrappers over simple lookups (team index ↔ name/color). Exporting them as functions (rather than macros or inline) allows implementations to be changed later without recompiling callers — useful for runtime i18n or team name customization.

## Data Flow Through This File

**Flag Pickup & Return Loop:**
```
Game entity (flag) spawned
  → Team_InitGame() inits global flag state
  → Player movement triggers Pickup_Team(flag, player)
    → Attaches flag to player; sets carrier state
  → If flag dropped: Team_DroppedFlagThink() called each frame
    → Decrements CTF_FLAG_RETURN_TIME countdown
    → At timeout: Team_ReturnFlag(team) auto-returns to base
```

**Frag Bonus Calculation:**
```
G_Damage(victim, attacker, ...) called in g_combat.c
  → At death: Team_FragBonuses(victim, attacker)
    → Uses trap_Trace or entity position queries to check:
      - Is attacker within CTF_ATTACKER_PROTECT_RADIUS of flag? → bonus
      - Is victim within CTF_TARGET_PROTECT_RADIUS of flag? → bonus
      - Did attacker recently harm flag carrier? (CTF_CARRIER_DANGER_PROTECT_TIMEOUT) → bonus
    → Calls AddTeamScore(origin, team, bonus_points)
      → Broadcasts team score event to all clients via configstring
      → Triggers client-side HUD refresh + optional world message
```

## Learning Notes

This file exemplifies several idiomatic Q3A patterns:

- **Macro-Heavy Configuration**: In the 2000s, build-time constants via preprocessor were the standard "configuration" mechanism. Modern engines use data files (YAML/JSON) + runtime loads, avoiding recompilation for balance tuning.

- **Explicit Constants Over Derived Values**: `CTF_FRAG_CARRIER_ASSIST_BONUS = 10` is hardcoded, not computed from damage dealt or other metrics. Q3A prioritizes **predictable, easy-to-tune rules** over emergent/dynamic scoring.

- **Entity-Centric Architecture**: All functions accept `gentity_t*` pointers. There is no "Team" object; team state lives in scattered globals and per-entity flags. This is simpler but less encapsulated than an ECS or OOP approach.

- **Proximity-Based Bonuses**: Rather than complex AI reasoning, bonuses reward players for **being near objectives**. This is a deliberate design choice: high skill ceiling (know where to be) without requiring AI-hard problem-solving.

## Potential Issues

1. **Hard-Coded Spatial/Temporal Constants**: All 1000-unit radii and 8–10 second timeouts are burned into the binary. Changing them requires recompilation and patch distribution. No way to tune gameplay post-release without shipping a new build. Modern servers use config files or engine console variables.

2. **Documentation Inconsistency**: `CTF_FLAG_RETURN_TIME` comment says "seconds until auto return" but the value `40000` is in **milliseconds** (40 seconds). `CTF_CARRIER_DANGER_PROTECT_TIMEOUT 8000` is also ms but comment doesn't clarify units. This invites bugs.

3. **Conditional Compilation Coupling**: Code in `g_team.c` must handle both `#ifdef MISSIONPACK` paths. If a third game variant were added (e.g., a softer balance), the branching logic in the .c file would become a maintenance burden. A runtime balance-set selector (loaded from config) would scale better.

4. **No Runtime Validation**: Nothing prevents a misconfigured balance set (e.g., all bonuses = 0) from being compiled. A validation function called at `Team_InitGame()` would catch obvious errors.

5. **Implicit Coupling to `g_local.h` Globals**: Functions like `Team_ReturnFlag()` likely read/write global flag entity pointers (`g_flag[TEAM_RED/TEAM_BLUE]` or similar). This implicit coupling is not visible in the header — callers don't know what state is being modified.

---

**Cross-Cutting Insight:** This header encapsulates the "game rules as configuration" philosophy that made Q3A easy to mod and tune. By contrast, more complex game logic (pathing, AI, damage falloff) lives in `.c` files with less exposure. Team/CTF rules are intentionally shallow so mappers and balance designers can tweak them without C expertise.
