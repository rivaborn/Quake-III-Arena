# code/botlib/be_aas_optimize.h

## File Purpose
Public header for the AAS (Area Awareness System) optimization module. It exposes a single entry point used to post-process and optimize AAS world data after loading or compilation.

## Core Responsibilities
- Declare the public interface for AAS optimization
- Expose `AAS_Optimize` as the sole external entry point for AAS data compaction/cleanup

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_Optimize
- Signature: `void AAS_Optimize(void);`
- Purpose: Performs optimization passes on the loaded AAS world data (e.g., merging redundant reachabilities, pruning unused nodes/areas).
- Inputs: None (operates on globally held AAS world state, defined in `be_aas_def.h`/`be_aas_main.c`).
- Outputs/Return: `void`
- Side effects: Modifies global AAS world data structures in-place; may reallocate or compact internal arrays.
- Calls: Defined in `be_aas_optimize.c`; called externally by AAS initialization/load code (likely `AAS_LoadAASFile` or equivalent in `be_aas_file.c` / `be_aas_main.c`).
- Notes: No parameters — relies entirely on shared AAS global state; must be called after AAS data is fully loaded and before routing/pathfinding begins.

## Control Flow Notes
Called once during AAS initialization, after world data is loaded from disk and before the bot routing tables are computed. It is not invoked per-frame. Fits into the init phase of the botlib lifecycle.

## External Dependencies
- No includes in this header.
- `AAS_Optimize` is defined in `code/botlib/be_aas_optimize.c` (defined elsewhere).
- Consumers: AAS load/init routines in `be_aas_main.c` or `be_aas_file.c`.
