# code/botlib/be_ai_move.c

## File Purpose
Implements the bot movement AI for Quake III Arena, translating high-level goal navigation into frame-by-frame elementary actions (EA_Move, EA_Jump, etc.) using the AAS (Area Awareness System) reachability graph. It manages per-bot movement state and handles all travel types from walking and jumping to grappling hooks and weapon jumps.

## Core Responsibilities
- Allocate, initialize, and free per-bot `bot_movestate_t` instances
- Determine which AAS reachability area a bot currently occupies
- Select the next reachability link toward a goal via routing and avoid-spot filtering
- Execute travel-type-specific movement logic (walk, crouch, jump, ladder, elevator, grapple, rocket/BFG jump, jump pad, func_bobbing, teleport, water jump)
- Manage reachability timeout, avoid-reach blacklisting, and avoid-spot hazard detection
- Initialize/shutdown the move AI subsystem, registering libvars for physics constants

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_movestate_t` | struct | Per-bot movement state: origin, velocity, flags, area tracking, avoid-reach arrays, avoid-spots |
| `bot_moveresult_t` | struct (defined in `be_ai_move.h`) | Output of a movement step: direction, travel type, failure/blocked flags, weapon/view overrides |
| `bot_initmove_t` | struct (defined in `be_ai_move.h`) | Input snapshot used to initialize a move state each frame |
| `bot_avoidspot_t` | struct (defined in `be_ai_move.h`) | Hazard zone (origin + radius + type) the bot should route around |
| `aas_reachability_t` | struct (defined in AAS headers) | A directed edge in the AAS graph: start/end positions, travel type, face/edge/area data |
| `libvar_t` | struct | Botlib configuration variable (sv_gravity, sv_maxstep, weapon indices, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botmovestates` | `bot_movestate_t *[MAX_CLIENTS+1]` | global | Per-bot move state pool, indexed by handle (1-based) |
| `modeltypes` | `int[MAX_MODELS]` | global | Maps BSP model index → MODELTYPE_* constant for mover classification |
| `sv_maxstep` | `libvar_t *` | global | Step height limit (physics constant) |
| `sv_maxbarrier` | `libvar_t *` | global | Barrier jump height limit |
| `sv_gravity` | `libvar_t *` | global | Gravity value for air-control prediction |
| `weapindex_rocketlauncher` | `libvar_t *` | global | Weapon index used for rocket jumps |
| `weapindex_bfg10k` | `libvar_t *` | global | Weapon index for BFG jumps |
| `weapindex_grapple` | `libvar_t *` | global | Grapple weapon index |
| `entitytypemissile` | `libvar_t *` | global | Entity type value for missile entities |
| `offhandgrapple` | `libvar_t *` | global | Whether grapple is off-hand (changes activation commands) |
| `cmd_grappleon/off` | `libvar_t *` | global | Console command strings for grapple activation |

## Key Functions / Methods

### BotSetupMoveAI
- Signature: `int BotSetupMoveAI(void)`
- Purpose: Initializes the move AI subsystem at bot library startup.
- Inputs: None
- Outputs/Return: `BLERR_NOERROR`
- Side effects: Populates all global `libvar_t *` pointers; calls `BotSetBrushModelTypes()`.
- Calls: `BotSetBrushModelTypes`, `LibVar`

### BotShutdownMoveAI
- Signature: `void BotShutdownMoveAI(void)`
- Purpose: Frees all allocated move states on shutdown.
- Side effects: Frees and NULLs all `botmovestates[1..MAX_CLIENTS]`.

### BotMoveToGoal
- Signature: `void BotMoveToGoal(bot_moveresult_t *result, int movestate, bot_goal_t *goal, int travelflags)`
- Purpose: Main per-frame movement entry point. Determines the bot's current AAS area, selects or reuses a reachability link toward the goal, and dispatches to the appropriate `BotTravel_*` or `BotFinishTravel_*` handler.
- Inputs: `result` (output struct), `movestate` handle, `goal`, allowed `travelflags`
- Outputs/Return: Fills `*result`; no return value.
- Side effects: Updates `ms->lastreachnum`, `ms->lastareanum`, `ms->lastgoalareanum`, `ms->reachability_time`, `ms->jumpreach`, `ms->moveflags`; may call EA_* action functions.
- Calls: `BotResetGrapple`, `BotOnTopOfEntity`, `BotFuzzyPointReachabilityArea`, `BotMoveInGoalArea`, `BotGetReachabilityToGoal`, `BotAddToAvoidReach`, `BotTravel_*`, `BotFinishTravel_*`, `AAS_*` queries
- Notes: Handles the in-air case separately (jump pad detection, finish-travel dispatch). Accelerates timeout when `result->blocked`.

### BotGetReachabilityToGoal
- Signature: `int BotGetReachabilityToGoal(vec3_t origin, int areanum, int lastgoalareanum, int lastareanum, int *avoidreach, float *avoidreachtimes, int *avoidreachtries, bot_goal_t *goal, int travelflags, int movetravelflags, struct bot_avoidspot_s *avoidspots, int numavoidspots, int *flags)`
- Purpose: Iterates outgoing reachabilities from `areanum`, filters by avoid-reach blacklist and avoid-spots, queries routing cost to goal, and returns the best reachability number.
- Outputs/Return: Best reachability number, or 0 if none found.
- Calls: `AAS_NextAreaReachability`, `AAS_ReachabilityFromNum`, `BotValidTravel`, `AAS_AreaTravelTimeToGoalArea`, `BotAvoidSpots`

### BotTravel_Walk / BotFinishTravel_Walk
- Purpose: Generate EA_Move commands for walking toward a reachability start/end point; handles gap detection, crouch areas, and walk-speed reduction near gaps.
- Calls: `BotCheckBlocked`, `BotGapDistance`, `EA_Move`, `EA_Walk`, `EA_Crouch`

### BotTravel_Jump / BotFinishTravel_Jump
- Purpose: Approach a run-up point then trigger `EA_Jump` at the correct moment; during air phase, maintain speed toward landing.
- Calls: `AAS_JumpReachRunStart`, `EA_Jump`, `EA_DelayedJump`, `EA_Move`

### BotTravel_Grapple / BotResetGrapple / GrappleState
- Purpose: Full grapple hook lifecycle: approach start, aim, fire, track hook state, detect stall, cancel on timeout.
- Calls: `GrappleState`, `EA_Command`, `EA_Attack`, `AAS_Trace`, `AngleDiff`
- Notes: Handles both off-hand and weapon-slot grapple modes.

### BotTravel_RocketJump / BotTravel_BFGJump
- Purpose: Align view downward, jump + attack simultaneously to exploit splash damage for vertical mobility.
- Side effects: `EA_Jump`, `EA_Attack`, `EA_SelectWeapon`, `EA_View`; sets `result.flags |= MOVERESULT_MOVEMENTWEAPON | MOVERESULT_MOVEMENTVIEWSET`.

### BotTravel_Elevator / BotFinishTravel_Elevator
- Purpose: Wait for a `func_plat` to descend, board it, ride to top, then step off toward the reachability end.
- Calls: `BotOnMover`, `MoverDown`, `MoverBottomCenter`, `EA_Move`, `BotCheckBarrierJump`

### BotWalkInDirection
- Signature: `int BotWalkInDirection(bot_movestate_t *ms, vec3_t dir, float speed, int type)`
- Purpose: Uses `AAS_PredictClientMovement` to validate a move before committing; detects gaps, slime/lava, and blocking.
- Calls: `AAS_PredictClientMovement`, `BotCheckBarrierJump`, `BotGapDistance`, `EA_Jump`, `EA_Crouch`, `EA_Move`

### BotFuzzyPointReachabilityArea
- Signature: `int BotFuzzyPointReachabilityArea(vec3_t origin)`
- Purpose: Finds the nearest AAS area with reachability links to a given world point, sampling a 3×3×3 grid with small offsets to handle floating-point boundary cases.
- Calls: `AAS_PointAreaNum`, `AAS_AreaReachability`, `AAS_TraceAreas`

### BotReachabilityArea
- Signature: `int BotReachabilityArea(vec3_t origin, int client)`
- Purpose: Classifies what the bot is standing on (world, func_plat, func_bob, other entity) and returns the appropriate reachability area number.
- Calls: `AAS_Trace`, `AAS_EntityModelindex`, `AAS_OriginOfMoverWithModelNum`, `AAS_NextModelReachability`, `AAS_Swimming`, `BotFuzzyPointReachabilityArea`

### BotSetBrushModelTypes
- Purpose: Iterates all BSP entities and classifies model indices into `modeltypes[]` (plat, bob, door, static).
- Side effects: Writes to `modeltypes[]` global array.

## Control Flow Notes
- **Init**: `BotSetupMoveAI` called once at botlib init; registers libvars and classifies brush models.
- **Per-frame**: `BotInitMoveState` updates the move state from server-supplied snapshot, then `BotMoveToGoal` is called to produce movement output.
- **Dispatch**: `BotMoveToGoal` branches on `MFL_ONGROUND | MFL_SWIMMING | MFL_AGAINSTLADDER` vs. in-air, then switch-dispatches on `reach.traveltype` to ~15 specialized travel handlers.
- **Shutdown**: `BotShutdownMoveAI` frees all move state allocations.

## External Dependencies
- **AAS API** (`be_aas_funcs.h`, `be_aas.h`): `AAS_PointAreaNum`, `AAS_TraceAreas`, `AAS_TraceClientBBox`, `AAS_Trace`, `AAS_PredictClientMovement`, `AAS_ReachabilityFromNum`, `AAS_NextAreaReachability`, `AAS_AreaReachability`, `AAS_AreaTravelTimeToGoalArea`, `AAS_TravelFlagForType`, `AAS_Swimming`, `AAS_OnGround`, `AAS_AgainstLadder`, `AAS_JumpReachRunStart`, `AAS_OriginOfMoverWithModelNum`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_NextModelReachability`, `AAS_EntityModelindex`, `AAS_EntityModelNum`, `AAS_EntityInfo`, `AAS_EntityType`, `AAS_NextEntity`, `AAS_Time`, `AAS_PointContents`, `AAS_HorizontalVelocityForJump`, `AAS_AreaPresenceType`, `AAS_AreaContentsTravelFlags`, `AAS_AreaDoNotEnter`, `AAS_AreaJumpPad`, `AAS_NextBSPEntity`, `AAS_ValueForBSPEpairKey`
- **EA API** (`be_ea.h`): `EA_Move`, `EA_Jump`, `EA_DelayedJump`, `EA_Crouch`, `EA_Walk`, `EA_Attack`, `EA_MoveForward`, `EA_MoveUp`, `EA_View`, `EA_SelectWeapon`, `EA_Command`
- **botimport**: `Print` for debug/error messages (defined in `be_interface.h`)
- **bot_developer**: External debug flag (defined elsewhere)
- `GetClearedMemory` / `FreeMemory`: from `l_memory.h`
- `LibVar`: from `l_libvar.h`
