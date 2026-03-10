# code/bspc/aas_facemerging.h

## File Purpose
Public interface header for the AAS face merging subsystem within the BSPC (BSP Compiler) tool. Declares two functions responsible for merging coplanar faces within AAS areas to reduce geometry complexity during AAS file generation.

## Core Responsibilities
- Exposes the face merging API to other BSPC compilation units
- Declares area-level face merging (all faces across areas)
- Declares plane-constrained face merging (merging faces sharing the same plane)

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_MergeAreaFaces
- **Signature:** `void AAS_MergeAreaFaces(void)`
- **Purpose:** Merges faces within AAS areas, likely consolidating coplanar or adjacent face geometry across all areas to simplify the AAS spatial representation.
- **Inputs:** None (operates on global AAS world state)
- **Outputs/Return:** `void`
- **Side effects:** Modifies global AAS area/face data structures in place
- **Calls:** Not inferable from this file
- **Notes:** Implementation resides in `aas_facemerging.c`

### AAS_MergeAreaPlaneFaces
- **Signature:** `void AAS_MergeAreaPlaneFaces(void)`
- **Purpose:** Merges faces that share the same plane within each AAS area, reducing the total face count by combining coplanar polygons into single larger faces.
- **Inputs:** None (operates on global AAS world state)
- **Outputs/Return:** `void`
- **Side effects:** Modifies global AAS area/face data structures in place
- **Calls:** Not inferable from this file
- **Notes:** Likely called after `AAS_MergeAreaFaces` or as a distinct optimization pass targeting plane-coincident faces specifically

## Control Flow Notes
Both functions are invoked during the BSP-to-AAS compilation pipeline, as part of a post-processing/optimization phase after initial AAS area and face data has been constructed. They reduce geometric redundancy before the AAS data is written to disk. Callers are expected to be in `aas_create.c` or `bspc.c` driving the overall compilation sequence.

## External Dependencies
- No includes in this header
- Both declared functions are **defined in** `code/bspc/aas_facemerging.c`
- Implicitly depends on global AAS world state structures defined in `aas_create.h` / `aas_store.h` (not visible here)
