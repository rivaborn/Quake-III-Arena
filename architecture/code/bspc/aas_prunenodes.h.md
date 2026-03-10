# code/bspc/aas_prunenodes.h

## File Purpose
Public header for the AAS node pruning subsystem within the BSPC (BSP Compiler) tool. It declares a single entry point used during AAS (Area Awareness System) tree post-processing to remove unnecessary or degenerate nodes from the compiled BSP/AAS tree.

## Core Responsibilities
- Exposes `AAS_PruneNodes` for use by other BSPC compilation stages that need to invoke node pruning on the AAS tree.

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_PruneNodes
- Signature: `void AAS_PruneNodes(void);`
- Purpose: Prunes redundant, empty, or otherwise invalid nodes from the AAS BSP tree during the offline map compilation phase.
- Inputs: None (operates on shared global AAS/BSP tree state implicitly).
- Outputs/Return: `void`
- Side effects (global state, I/O, alloc): Modifies the global in-memory AAS node tree; may free pruned node memory; no direct I/O from this declaration alone.
- Calls: Not inferable from this file (declaration only).
- Notes: Implementation resides in `code/bspc/aas_prunenodes.c`. This is a compile-time tool function, not a runtime game function.

## Control Flow Notes
Called during the BSPC offline AAS compilation pipeline, after initial AAS area/node generation (e.g., after `AAS_Create`) and before AAS file output. It is a batch processing step with no per-frame relevance.

## External Dependencies
- No includes in this header.
- `AAS_PruneNodes` is defined in `code/bspc/aas_prunenodes.c` (defined elsewhere).
