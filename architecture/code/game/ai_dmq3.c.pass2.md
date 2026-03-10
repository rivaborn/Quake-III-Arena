# code/game/ai_dmq3.c — Enhanced Analysis

## Architectural Role

This file is the **core bot decision-making layer** for Quake III Arena deathmatch and team modes. It bridges the per-frame game loop (driven by `ai_main.c`) to the botlib navigation/AI library below, and acts as the integration point for all bot perception (enemy detection, visibility, item awareness) and tactical reasoning (long-term goal selection, combat readiness, obstacle recovery). It is the primary consumer of botlib's AAS pathfinding and movement prediction APIs, and the primary producer of `usercmd_t` input state that the game processes as if from a human player.

## Key Cross-References

### Incoming (who depends on this file)
- **`ai_main.c:BotAIStartFrame`** calls `BotDeathmatchAI(bs, thinktime)` once per frame per active bot
- **`ai_dmnet.c`** (team AI and node switching) calls into several `BotCTFSeekGoals`, `BotFindEnemy`, `BotAimAtEnemy`, `BotCheckAttack` as part of the larger AI FSM orchestrated by `AINode_*` state functions
- **`g_bot.c`** calls `BotSetupDeathmatchAI` at map load to initialize module globals (`gametype`, `maxclients`, flag positions)
- **`g_bot.c`** calls `BotShutdownDeathmatchAI` on map unload
- **Other game modules** (`g_client.c`, `g_combat.c`, `g_items.c`) indirectly depend on inventory sync via `BotUpdateInventory` to ensure bot AI perceives the correct item/weapon state

### Outgoing (what this file depends on)
- **botlib (`code/botlib/be_*.h`)**:
  - `trap_AAS_AreaTravelTimeToGoalArea` — path distance estimation
  - `trap_AAS_PointAreaNum` — map area lookup from position
  - `trap_AAS_EntityInfo`, `trap_AAS_AreaInfo` — navigation mesh queries
  - `trap_AAS_PredictClientMovement` — jump arc prediction for aim lead
  - `trap_BotChooseLTGoal`, `trap_BotChooseNBGoal` — fuzzy-logic goal selection
  - `trap_BotGetWeaponInfo` — weapon properties (projectile speed, gravity, spread)
  - `trap_Characteristic_BFloat` — per-bot personality trait sampling (aim, aggression, etc.)

- **game module (`code/game/g_local.h`, `g_*.c`)**:
  - `BotAI_GetClientState`, `BotAI_GetEntityState` — player/entity snapshot reads
  - `BotEntityInfo` — reads aas entity snapshots
  - `trap_GetConfigstring`, `trap_SetUserinfo`, `Info_SetValueForKey` — team/status queries
  - `trap_GetUserinfo` — player config read for team detection
  - `G_ModelIndex` — BSP entity model lookup (via `max_bspmodelindex`)

- **AI chat/team modules** (`ai_chat.h`, `ai_dmnet.h`, `ai_team.h`)**:
  - `BotVoiceChat` — audio communication
  - `BotChat_EnterGame` — join announcements
  - `BotTeamAI` — higher-level team goal coordination
  - `BotSameTeam` — team affiliation checks
  - `BotSetTeamStatus` — broadcast current task to teammates
  - `AIEnter_Seek_LTG`, `AIEnter_Stand`, `AIEnter_Seek_ActivateEntity` — FSM node transitions

- **Shared utilities**:
  - `FloatTime`, `NumBots` — clock and bot enumeration
  - `BotAI_Trace`, `BotAI_Print` — line-of-sight traces and debug logging
  - `trap_EA_*` family — EA (Elementary Action) input queuing (`trap_EA_View`, `trap_EA_Attack`, `trap_EA_Move`)

## Design Patterns & Rationale

### 1. **Deferred Per-Bot Setup**
`setupcount` throttles initialization work over multiple frames. This avoids the frame spike that would occur if all bots loaded their inventories, snapshots, and waypoints simultaneously. Modern engines would batch this, but for a late-90s engine running 8–16 bots, frame-spreading is pragmatic.

### 2. **Visibility with Fog Attenuation**
`BotEntityVisible` implements a physically-plausible visibility model by:
- Tracing to origin + 3 sample points (bottom, center, top) to handle tall targets and partial occlusion
- Returning a float [0, 1] visibility factor based on water/fog distance, allowing "dim" sighting at distance
- This contrasts with binary trace results and enables richer aiming behavior (shooting at "suspected" positions)

### 3. **Predictive Aiming with Skill Falloff**
`BotAimAtEnemy` uses characteristic-driven skill interpolation:
- High skill (>0.8): full physics-based projectile prediction via `AAS_PredictClientMovement`
- Medium skill (0.4–0.8): linear lead only
- Low skill (<0.4): no prediction, raw aim noise
This is a clean way to make bots feel less superhuman without branching code.

### 4. **Reflexive Attack Gating**
`BotCheckAttack` implements reaction-time delay via `characteristic`, weapon-cooldown tracking, and friendly-fire checks. This mirrors how human players have latency and decision delays. The weapon-change penalty is tracked in `weaponchange_time` to prevent spam-switching.

### 5. **Global Flag/Obelisk Cache**
`ctf_redflag`, `ctf_blueflag`, etc. are pre-fetched at map load and stored globally. This avoids repeated goal lookups on every frame and is safe because objectives don't move during a map's lifetime. (Compare to a more dynamic system where goals might be data-driven.)

### 6. **Alternative Route Goals for Team Modes**
`red_altroutegoals` and `blue_altroutegoals` are pre-computed waypoints used when the direct path to the enemy base is slower than the base defense route. This allows tactical route selection without full recomputation.

### 7. **Waypoint Pool with Free-List**
`botai_waypoints[]` and `botai_freewaypoints` use a classic Quake-era linked-list allocator. Allows lightweight waypoint creation without malloc overhead. The pool is static (128 max), so it's never realloc'd.

## Data Flow Through This File

### **Per-Frame (BotDeathmatchAI)**
```
1. Input: frame tick, bot_state_t with old inventory/snapshot
   ↓
2. Lazy Setup (setupcount frames): load inventory, snapshot, initial waypoints
   ↓
3. Update: reflect latest playerState_t → inventory[]; detect new items
   ↓
4. Perception: BotCheckSnapshot (entity state), BotCheckAir (falling), console msgs
   ↓
5. Decision: BotTeamAI (long-term goal: CTF, defend, obelisk) → ltgtype / teamgoal
   ↓
6. Node Loop (repeat until max switches or state stable):
   - Run current FSM node (AINode_Seek_LTG, AINode_Battle, etc.)
   - May call BotFindEnemy, BotAimAtEnemy, BotCheckAttack
   - May transition to new node (AIEnter_*)
   ↓
7. Output: queued EA input (movement, aim, attack) → trap_EA_*
```

### **Enemy Perception (BotFindEnemy)**
```
Input: current bot state, last known enemy
  ↓
1. Scan all clients (MAX_CLIENTS loop)
2. Per client: check dead? invisible? friendly fire? 
   ↓
3. Visibility test: BotEntityVisible (with fog attenuation)
   ↓
4. Range/FOV check: distance < max?, within expanded FOV?
5. Threat scoring: quad? carrying flag? shooting?
   ↓
Output: best enemy index, update enemysight_time
```

### **Aim Prediction (BotAimAtEnemy)**
```
Input: enemy position/velocity, weapon info, bot skill
  ↓
1. Fetch target entity info (origin, velocity, bounds)
2. Weapon speed check: is it hitscan or projectile?
3. If projectile + skill > 0.8: AAS_PredictClientMovement(enemy state)
   → compute enemy position at lead time
4. Else: linear lead (aimtarget = enemy_pos + enemy_vel * lead_time)
5. Add skill-based noise to viewangles
6. Challenge mode: also call trap_EA_View for determinism
   ↓
Output: bs->ideal_viewangles toward lead point
```

### **Blocked Movement (BotAIBlocked)**
```
Input: moveresult with MOVERESULT_BLOCKED
  ↓
1. Check if blocking entity is activatable (button, door, trigger)
2. If activate=true && blocking entity found:
   → BotGoForActivateGoal(entity) to queue button-press goal
3. Else: attempt evasion (strafe left/right, crouch)
4. If blocked > threshold: abandon current goal and re-plan
   ↓
Output: new goal or movement command
```

## Learning Notes

### What a Developer Studying This File Learns

1. **Integration of AI into a Server-Authoritative Game**  
   The bot is not a separate entity; it's a client peer to humans. Its input is synthesized via the same `usercmd_t` pipeline, routed through the same `Pmove`, and subject to the same lag compensation. This teaches how to retrofit AI into an existing engine without breaking server authority.

2. **Visibility and Perception at 3D Scale**  
   `BotEntityVisible` with fog attenuation is a simple yet effective perception model. It accounts for:
   - Geometric occlusion (traces)
   - Environmental visibility (fog/water distance)
   - Partial targets (multi-sample tracing)
   
   This is more nuanced than simple "can I trace to the enemy?" but simpler than full line-of-sight cone tracing. Useful template for other games.

3. **Skill-Based Aiming Without Separate Code Paths**  
   The characteristic-driven falloff (`aim_skill` ∈ [0,1] → linear/physics/none prediction) is elegant: one code path, continuous difficulty spectrum, no "easy/medium/hard" hardcoding. Modern game AI often struggles with this.

4. **Team Game Mode Dispatch via Global State**  
   `BotCTFSeekGoals`, `BotObeliskSeekGoals`, etc. are separate functions dispatched from `BotTeamAI` based on `gametype`. This is the 90s pattern before data-driven gameplay. Modern engines would have a data table, but the logic separation is still clean.

5. **FSM Node Switching as a Throttle**  
   The `MAX_NODESWITCHES` limit prevents pathological FSM loops (e.g., infinite transitions between Battle and Seek). If a bot is switching nodes on every frame, something is wrong and it gets logged. This is defensive programming.

### What's Idiomatic to This Era / Engine

- **No heap allocations in hot paths**: waypoints use a static pool, goals are stack-allocated `bot_goal_t` structs
- **Global caching of static data**: flag positions, max BSP model index computed once at map load
- **Characteristic-based scripting**: bot personality is a float per trait, interpolated at runtime from a text database (not shown here but called via `trap_Characteristic_BFloat`)
- **Lightweight physics prediction**: `AAS_PredictClientMovement` simulates only gravity + friction, not full client-side move code (that's expensive)
- **Trace-based line of sight**: no portal/PVS overhead, just raycasts; works on small 1v1 maps but would scale poorly to open-world

### Connections to Modern Engine Concepts

- **ECS-like organization**: `bot_state_t` is a monolithic entity state bag (old pattern), but callsites like `BotDeathmatchAI` iterate over `bot_state_t *` array, which resembles iterating an archetype
- **Behavior Tree flavor**: `AINode_*` functions are state-machine callbacks, not tree nodes, but the layered dispatch (team AI → long-term → combat) mirrors behavior tree depth
- **GOAP-like goal selection**: `trap_BotChooseLTGoal` (called via `BotTeamAI`) uses fuzzy-logic scoring, not GOAP, but the intent is similar: rank candidate goals by state and pick best

## Potential Issues

1. **Visibility Fog Attenuation Edge Case**  
   `BotEntityVisible` returns a float visibility factor based on fog distance. If fog is disabled (distance = 0), the function may return spurious non-zero values due to uninitialized distance. Should check `FP_ZERO` or clamp defensively.

2. **Aim Prediction Assumes Constant Velocity**  
   `BotAimAtEnemy` uses `AAS_PredictClientMovement`, which is expensive and only called for high-skill bots. For medium-skill, it falls back to linear lead using enemy velocity. But if the enemy is accelerating or jumping, the lead is wrong. Acceptable for an old bot, but worth documenting.

3. **Node Switch Infinite Loop Detection**  
   The code logs an error if `MAX_NODESWITCHES` is hit, but doesn't forcibly break out with a default node. If an FSM bug causes infinite transitions, the bot hangs for that frame (and subsequent frames). A safety fallback to `AIEnter_Stand` would be more graceful.

4. **Race Condition in Flag Status Polling**  
   `redflagstatus` and `blueflagstatus` are set by the game module asynchronously (via game entity events). If a bot checks status between event processing and its own update, it might make decisions on stale flag state. Not a crash, but can cause illogical behavior (e.g., "return flag" when it's already at base). Mitigated by checking actual flag entity position as a fallback.

5. **Waypoint Pool Exhaustion**  
   `MAX_WAYPOINTS = 128` is fixed. If more than 128 simultaneous waypoints are needed (rare but possible in complex maps with many bots and alternate routes), `botai_freewaypoints` will be NULL and `BotCreateWayPoint` will return NULL. Callers should check for NULL, but not all do. Should either increase the limit or add graceful degradation.
