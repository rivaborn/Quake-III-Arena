# code/bspc/be_aas_bspc.h

## File Purpose
Header file for the BSPC (BSP Compiler) AAS integration layer. It declares the single entry point used by the BSPC tool to trigger full AAS (Area Awareness System) reachability and cluster computation from a compiled BSP map file.

## Core Responsibilities
- Exposes `AAS_CalcReachAndClusters` as the public interface between the BSPC compilation pipeline and the AAS subsystem.

## Key Types / Data Structures
None. (References `struct quakefile_s` by forward declaration only; defined elsewhere.)

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_CalcReachAndClusters
- **Signature:** `void AAS_CalcReachAndClusters(struct quakefile_s *qf);`
- **Purpose:** Triggers full AAS post-processing on a compiled BSP map: calculates reachability links between AAS areas and partitions those areas into clusters for hierarchical pathfinding.
- **Inputs:** `qf` — pointer to a `quakefile_s` describing the source BSP/map file to process.
- **Outputs/Return:** `void`
- **Side effects:** Writes results into the global AAS world state (reachability arrays, cluster assignments); not inferable in full detail from this file alone.
- **Calls:** Not inferable from this file.
- **Notes:** The use of a struct forward declaration (not a full `#include`) keeps this header lightweight and avoids a circular dependency with the quakefile type.

## Control Flow Notes
Called during the BSPC offline map-compilation pipeline, after BSP geometry has been converted to AAS areas. This is a batch/offline step, not part of the runtime game loop.

## External Dependencies
- `struct quakefile_s` — defined elsewhere (likely `code/bspc/qfiles.h` or a shared BSP header); only forward-declared here.
