# code/bspc/aas_edgemelting.h

## File Purpose
Header file declaring the public interface for the AAS edge-melting pass within the BSPC (BSP compiler) tool. It exposes a single function used to simplify area face geometry by merging redundant collinear edges in AAS area windings.

## Core Responsibilities
- Declares `AAS_MeltAreaFaceWindings`, the sole public entry point for the edge-melting subsystem
- Acts as the module boundary between the edge-melting implementation (`aas_edgemelting.c`) and the rest of the BSPC pipeline

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_MeltAreaFaceWindings
- **Signature:** `void AAS_MeltAreaFaceWindings(void);`
- **Purpose:** Iterates over all AAS area face windings and removes colinear or redundant vertices/edges, "melting" them together to produce cleaner polygon geometry before final AAS file output.
- **Inputs:** None (operates on globally shared AAS build state)
- **Outputs/Return:** `void`
- **Side effects:** Mutates global AAS area/face/winding data structures in place; no I/O or allocation visible from this declaration alone.
- **Calls:** Not inferable from this file.
- **Notes:** The no-argument signature implies all state is accessed via global or module-static structures maintained by the BSPC AAS construction pipeline.

## Control Flow Notes
This function is called as a post-processing step during the BSPC AAS compilation pipeline, after initial area and face geometry has been generated (e.g., after `aas_create`/`aas_facemerging` passes) and before the AAS data is written to disk. It is a one-shot batch pass, not called per-frame.

## External Dependencies
- No includes in this header; the implementation (`aas_edgemelting.c`) depends on shared BSPC AAS internal structures (areas, faces, windings) defined elsewhere in the `bspc/` subsystem.
- `AAS_MeltAreaFaceWindings` — defined in `code/bspc/aas_edgemelting.c`
