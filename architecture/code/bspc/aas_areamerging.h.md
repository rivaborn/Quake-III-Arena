# code/bspc/aas_areamerging.h

## File Purpose
Header file declaring the public interface for the AAS (Area Awareness System) area merging pass within the BSPC (BSP Compiler) tool. It exposes a single entry-point function used during AAS world generation to reduce area count by combining adjacent compatible areas.

## Core Responsibilities
- Declares the `AAS_MergeAreas` function as a public symbol for use by other BSPC compilation units

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_MergeAreas
- Signature: `void AAS_MergeAreas(void)`
- Purpose: Triggers a pass over the AAS area graph to merge adjacent areas that share compatible properties, reducing total area count and simplifying the navigation mesh.
- Inputs: None (operates on globally shared AAS world state)
- Outputs/Return: `void`
- Side effects: Mutates global AAS area/face/edge data structures in-place; defined in `aas_areamerging.c`
- Calls: Not inferable from this file
- Notes: No parameters — relies entirely on module-level global state populated by prior BSP/AAS build stages

## Control Flow Notes
Called as a discrete compilation pass within the BSPC tool's AAS generation pipeline, after initial AAS areas have been created from the BSP tree (e.g., after `AAS_CreateAreas`) and before final area optimization or file output. It is a batch, offline operation — not part of the runtime game loop.

## External Dependencies
- No includes in this header; implementation dependencies are in `aas_areamerging.c`
- `AAS_MergeAreas` is defined elsewhere (`aas_areamerging.c`)
