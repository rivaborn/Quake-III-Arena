# code/botlib/be_aas_reach.h

## File Purpose
Public and internal interface header for the AAS (Area Awareness System) reachability subsystem of Quake III Arena's bot library. It declares functions for querying area traversal properties and computing reachability relationships between AAS areas.

## Core Responsibilities
- Declare initialization and incremental computation of area reachabilities (internal only)
- Expose area property queries (swim, liquid, lava, slime, crouch, grounded, ladder, jump pad, do-not-enter)
- Provide spatial queries to find the best reachable area from a given origin/bounding box
- Support jump pad reachability queries
- Provide model-based reachability iteration

## Key Types / Data Structures
None defined here; relies on types defined elsewhere (`vec3_t`, `aas_link_t`).

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_InitReachability *(AASINTERN)*
- Signature: `void AAS_InitReachability(void)`
- Purpose: Begins the reachability calculation pass for all AAS areas.
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies internal AAS world state; defined in `be_aas_reach.c`
- Calls: Not inferable from this file
- Notes: Guarded by `#ifdef AASINTERN`; not exposed to game/cgame modules

### AAS_ContinueInitReachability *(AASINTERN)*
- Signature: `int AAS_ContinueInitReachability(float time)`
- Purpose: Incrementally advances reachability computation across frames to avoid stalls.
- Inputs: `time` — time budget (seconds) for this slice
- Outputs/Return: Non-zero when computation is complete
- Side effects: Mutates internal AAS reachability tables
- Notes: Supports amortized loading; guarded by `#ifdef AASINTERN`

### AAS_BestReachableArea
- Signature: `int AAS_BestReachableArea(vec3_t origin, vec3_t mins, vec3_t maxs, vec3_t goalorigin)`
- Purpose: Given a bounding box at an origin, finds the best AAS area that can be reached and outputs the goal origin within it.
- Inputs: `origin` — query point; `mins`/`maxs` — bounding box extents; `goalorigin` — out parameter for the reachable goal position
- Outputs/Return: Area number of the best reachable area; 0 or negative on failure
- Side effects: Writes `goalorigin`

### AAS_BestReachableFromJumpPadArea
- Signature: `int AAS_BestReachableFromJumpPadArea(vec3_t origin, vec3_t mins, vec3_t maxs)`
- Purpose: Finds the best jump pad area from which the given bounding box at `origin` is reachable.
- Inputs: `origin`, `mins`, `maxs` — bounding box description
- Outputs/Return: Area number of the relevant jump pad area

### AAS_NextModelReachability
- Signature: `int AAS_NextModelReachability(int num, int modelnum)`
- Purpose: Iterates reachabilities associated with a specific BSP model (e.g., doors, movers).
- Inputs: `num` — current iteration index; `modelnum` — BSP model number
- Outputs/Return: Next reachability index for the model; 0 when exhausted

### AAS_AreaGroundFaceArea
- Signature: `float AAS_AreaGroundFaceArea(int areanum)`
- Purpose: Returns the total surface area of all ground faces in an AAS area, useful for area weighting.
- Outputs/Return: Float area value in world units²

### Area Boolean Queries
- `AAS_AreaReachability(int areanum)` — has outgoing reachabilities
- `AAS_AreaCrouch(int areanum)` — crouch-only area
- `AAS_AreaSwim(int areanum)` — swimmable
- `AAS_AreaLiquid(int areanum)` — filled with any liquid
- `AAS_AreaLava(int areanum)` — contains lava
- `AAS_AreaSlime(int areanum)` — contains slime
- `AAS_AreaGrounded(int areanum)` — has ground faces
- `AAS_AreaLadder(int areanum)` — has ladder faces
- `AAS_AreaJumpPad(int areanum)` — is a jump pad
- `AAS_AreaDoNotEnter(int areanum)` — flagged as off-limits for routing

## Control Flow Notes
`AAS_InitReachability` is called during map load; `AAS_ContinueInitReachability` is called each server frame until complete. Area property queries are called at runtime during bot goal selection and path planning (from `be_aas_route.c` and `be_ai_goal.c`).

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_link_t` — defined in `be_aas_def.h` (internal AAS structure)
- Implementation: `be_aas_reach.c`
- Consumers: `be_aas_route.c`, `be_ai_goal.c`, `be_ai_move.c`
