# code/bspc/tetrahedron.h

## File Purpose
Header file declaring a single utility function for converting AAS (Area Awareness System) data into a tetrahedral representation. It serves as the public interface for `tetrahedron.c` within the BSPC (BSP Compiler) tool.

## Core Responsibilities
- Exposes one conversion function to other BSPC translation units
- Acts as the sole interface between AAS data and tetrahedral geometry output

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### TH_AASToTetrahedrons
- Signature: `void TH_AASToTetrahedrons(char *filename);`
- Purpose: Converts AAS file data into a tetrahedral mesh representation, likely for debugging or visualization of AAS volumes.
- Inputs: `filename` — path to the AAS file to read and convert.
- Outputs/Return: `void`; results are written to file or global state (not inferable from this header alone).
- Side effects: Almost certainly file I/O (reading AAS, writing output geometry); may allocate memory internally.
- Calls: Not inferable from this file.
- Notes: The `TH_` prefix is a module namespace convention used in BSPC. The exact output format (e.g., a debug mesh file) is not inferable from this file.

## Control Flow Notes
This header is included by other BSPC modules that need to trigger AAS-to-tetrahedron conversion. It is a build-time tool utility, not part of the runtime game engine; it runs during map compilation, not during gameplay frames.

## External Dependencies
- No includes in this header.
- `TH_AASToTetrahedrons` is defined in `code/bspc/tetrahedron.c` (defined elsewhere).
