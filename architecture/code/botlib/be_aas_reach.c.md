# code/botlib/be_aas_reach.c

## File Purpose
Computes all inter-area reachability links for the AAS (Area Awareness System) navigation graph. It classifies every possible movement transition between adjacent AAS areas (walk, jump, swim, ladder, teleport, elevator, etc.) and stores the results so the bot pathfinder can later query travel costs and start/end points.

## Core Responsibilities
- Allocate and manage a fixed-size heap of temporary `aas_lreachability_t` link objects during calculation.
- Detect and create reachability links for every movement type: swim, equal-floor walk, step, barrier jump, water jump, walk-off-ledge, jump, ladder, teleport, elevator, func_bobbing, jump pad, grapple hook, and weapon jump.
- Iterate over all area pairs across multiple frames (`AAS_ContinueInitReachability`) to spread CPU cost.
- Mark areas adjacent to high-value items as valid weapon-jump targets (`AAS_SetWeaponJumpAreaFlags`).
- Finalize calculation by converting linked `aas_lreachability_t` lists into the compact `aasworld.reachability` array via `AAS_StoreReachability`.

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_lreachability_t` | struct | Temporary linked-list node describing one directional movement link between two areas (start/end points, travel type, travel time). |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `reachabilityheap` | `aas_lreachability_t *` | global (file) | Pre-allocated pool of link nodes. |
| `nextreachability` | `aas_lreachability_t *` | global (file) | Next free node in the pool. |
| `areareachability` | `aas_lreachability_t **` | global (file) | Per-area linked list heads, one entry per AAS area. |
| `numlreachabilities` | `int` | global (file) | Count of currently allocated link nodes. |
| `reach_swim`, `reach_walk`, `reach_jump`, … (×19) | `int` | global | Per-type counters used for debug reporting. |
| `calcgrapplereach` | `int` | global | Flag: skip grapple reachability computation when 0. |

## Key Functions / Methods

### AAS_InitReachability
- **Signature:** `void AAS_InitReachability(void)`
- **Purpose:** Bootstraps the reachability calculation pass. Skips if data already exists and `forcereachability` is not set.
- **Inputs:** None (reads `aasworld`, lib vars).
- **Outputs/Return:** None.
- **Side effects:** Allocates `reachabilityheap`, `areareachability`; resets `aasworld.numreachabilityareas` to 1; calls `AAS_SetWeaponJumpAreaFlags`.
- **Calls:** `AAS_SetupReachabilityHeap`, `GetClearedMemory`, `AAS_SetWeaponJumpAreaFlags`, `LibVarGetValue`.
- **Notes:** Must be called once per map load before `AAS_ContinueInitReachability`.

### AAS_ContinueInitReachability
- **Signature:** `int AAS_ContinueInitReachability(float time)`
- **Purpose:** Incremental per-frame driver. Processes `REACHABILITYAREASPERCYCLE` areas per call and returns `qtrue` until finished.
- **Inputs:** `time` — unused directly; timing is via `Sys_MilliSeconds`.
- **Outputs/Return:** `qtrue` while still running; `qfalse` when complete.
- **Side effects:** Populates `areareachability[]`; on last step triggers `AAS_StoreReachability`, frees heap and `areareachability`.
- **Calls:** All `AAS_Reachability_*` functions, `AAS_StoreReachability`, `AAS_ShutDownReachabilityHeap`, `FreeMemory`, `Sys_MilliSeconds`.
- **Notes:** Call each server/bot frame until it returns `qfalse`.

### AAS_Reachability_Swim
- **Signature:** `int AAS_Reachability_Swim(int area1num, int area2num)`
- **Purpose:** Creates a `TRAVEL_SWIM` link when both areas are liquid and share a face.
- **Inputs:** Two area indices.
- **Outputs/Return:** `qtrue` if link created.
- **Side effects:** Allocates one `aas_lreachability_t`, prepends to `areareachability[area1num]`, increments `reach_swim`.
- **Calls:** `AAS_AreaSwim`, `AAS_FaceCenter`, `AAS_PointContents`, `AAS_AllocReachability`, `AAS_AreaVolume`.

### AAS_Reachability_EqualFloorHeight
- **Signature:** `int AAS_Reachability_EqualFloorHeight(int area1num, int area2num)`
- **Purpose:** Creates a `TRAVEL_WALK` link across a shared ground edge when the two areas are at the same height.
- **Inputs:** Two area indices.
- **Outputs/Return:** `qtrue` if link created.
- **Side effects:** Allocates one link, increments `reach_equalfloor`.
- **Calls:** `AAS_AreaGrounded`, `AAS_AllocReachability`, `AAS_AreaCrouch`.

### AAS_Reachability_Step_Barrier_WaterJump_WalkOffLedge
- **Signature:** `int AAS_Reachability_Step_Barrier_WaterJump_WalkOffLedge(int area1num, int area2num)`
- **Purpose:** Handles four related transition types depending on the vertical distance between adjacent ground/water edges: step-up (`TRAVEL_WALK`), water jump (`TRAVEL_WATERJUMP`), barrier jump (`TRAVEL_BARRIERJUMP`), walk-off-ledge (`TRAVEL_WALKOFFLEDGE`).
- **Inputs:** Two area indices.
- **Outputs/Return:** `qtrue` on first successful link.
- **Side effects:** Up to one link allocated; increments the relevant counter.
- **Calls:** `AAS_TraceClientBBox`, `AAS_TraceAreas`, `AAS_AreaClusterPortal`, `AAS_AllocReachability`, `AAS_FallDelta`, `AAS_AreaSwim`, `AAS_AreaJumpPad`.

### AAS_Reachability_Jump
- **Signature:** `int AAS_Reachability_Jump(int area1num, int area2num)`
- **Purpose:** Finds the closest pair of ground-face edges between two areas and simulates a predicted jump to verify `TRAVEL_JUMP` or `TRAVEL_WALKOFFLEDGE` connectivity.
- **Inputs:** Two area indices.
- **Outputs/Return:** `qfalse` always (link still created as side effect on success — `return qfalse` after link creation appears intentional to continue outer loops).
- **Side effects:** May allocate one link; increments `reach_jump` or `reach_walkoffledge`.
- **Calls:** `AAS_ClosestEdgePoints`, `AAS_PredictClientMovement`, `AAS_HorizontalVelocityForJump`, `AAS_TraceClientBBox`, `AAS_TraceAreas`.

### AAS_Reachability_Ladder
- **Signature:** `int AAS_Reachability_Ladder(int area1num, int area2num)`
- **Purpose:** Creates bidirectional `TRAVEL_LADDER` and `TRAVEL_WALKOFFLEDGE`/`TRAVEL_JUMP` links for areas sharing ladder faces.
- **Inputs:** Two area indices.
- **Outputs/Return:** `qtrue` if any links created.
- **Side effects:** Up to 2 links allocated; increments `reach_ladder` and possibly `reach_jump` or `reach_walkoffledge`.
- **Calls:** `AAS_AreaLadder`, `AAS_FaceArea`, `AAS_TraceClientBBox`, `AAS_ReachabilityExists`.

### AAS_Reachability_Teleport / AAS_Reachability_Elevator / AAS_Reachability_JumpPad / AAS_Reachability_FuncBobbing
- **Purpose (grouped):** Entity-driven passes that scan BSP entity key/value pairs to construct links for teleporters, func_plat elevators, trigger_push jump pads, and func_bobbing platforms.
- **Side effects:** Allocate links into `areareachability[]`; increment corresponding counters; call `botimport.Print` for diagnostics.
- **Notes:** These are only called once during the final consolidation step of `AAS_ContinueInitReachability`.

### AAS_StoreReachability
- **Signature:** `void AAS_StoreReachability(void)`
- **Purpose:** Converts per-area linked lists of `aas_lreachability_t` into the flat `aasworld.reachability[]` array consumed by the route planner.
- **Side effects:** Allocates `aasworld.reachability`; writes `areasettings[i].firstreachablearea` and `numreachableareas` for each area.
- **Calls:** `FreeMemory`, `GetClearedMemory`.

### AAS_SetupReachabilityHeap / AAS_ShutDownReachabilityHeap / AAS_AllocReachability / AAS_FreeReachability
- **Notes:** Pool allocator for `aas_lreachability_t`. Fixed capacity of `AAS_MAX_REACHABILITYSIZE` (65 536). `AAS_AllocReachability` returns `NULL` (and prints error) if the pool is exhausted.

## Control Flow Notes
- **Init phase:** `AAS_InitReachability` is called once after the AAS world is loaded. It prepares the heap and launches the incremental pass.
- **Update phase (per-frame):** `AAS_ContinueInitReachability` is called each bot frame until it returns `qfalse`. It processes a batch of area pairs per call, applying all geometric reachability detectors in priority order.
- **Finalization:** On the `numareas + 1` iteration, entity-based reachabilities (teleport, elevator, jump pad, func_bobbing, walk-off-ledge pass 2) are added, then `AAS_StoreReachability` converts everything to the permanent array and frees the heap.
- **Runtime query:** After finalization, `AAS_AreaReachability` and the `aasworld.reachability[]` array are used by `be_aas_route.c` for pathfinding; this file is not involved at runtime.

## External Dependencies
- `../game/q_shared.h` — math types, `qboolean`, vector macros
- `l_log.h`, `l_memory.h`, `l_libvar.h`, `l_precomp.h`, `l_struct.h` — botlib utilities
- `aasfile.h`, `be_aas_def.h` — `aasworld` global, AAS data structure definitions
- `../game/botlib.h`, `../game/be_aas.h`, `be_aas_funcs.h` — travel-type constants, presence types, BSP query APIs
- **Defined elsewhere:** `aasworld` (global singleton), `aassettings`, `botimport`, `AAS_TraceClientBBox`, `AAS_PredictClientMovement`, `AAS_ClientMovementHitBBox`, `AAS_HorizontalVelocityForJump`, `AAS_RocketJumpZVelocity`, `AAS_BFGJumpZVelocity`, `AAS_PointAreaNum`, `AAS_LinkEntityClientBBox`, `AAS_UnlinkFromAreas`, `AAS_TraceAreas`, `AAS_PointInsideFace`, `AAS_PointContents`, `AAS_AreaPresenceType`, `AAS_DropToFloor`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_NextBSPEntity`, `AAS_ValueForBSPEpairKey`, `AAS_VectorForBSPEpairKey`, `AAS_FloatForBSPEpairKey`, `AAS_IntForBSPEpairKey`, `AAS_PermanentLine`, `AAS_Trace`, `Sys_MilliSeconds`
