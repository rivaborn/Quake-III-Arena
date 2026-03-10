# code/bspc/aas_gsubdiv.h

## File Purpose
Declares two functions responsible for geometrically subdividing AAS (Area Awareness System) areas based on movement physics properties. This header is part of the BSPC (BSP Compiler) tool that converts BSP map data into AAS navigation data for bot pathfinding.

## Core Responsibilities
- Expose the gravitational subdivision pass interface for AAS area generation
- Expose the ladder subdivision pass interface for AAS area generation

## Key Types / Data Structures
None.

## Global / File-Static State
None declared here. The comment explicitly notes both functions operate on the global `tmpaasworld` (defined elsewhere in the BSPC tool).

## Key Functions / Methods

### AAS_GravitationalSubdivision
- **Signature:** `void AAS_GravitationalSubdivision(void)`
- **Purpose:** Subdivides AAS areas based on gravitational properties — likely splits areas at boundaries where gravity behavior changes (e.g., differentiating walkable floor regions from void/fall regions).
- **Inputs:** None (implicitly operates on global `tmpaasworld`)
- **Outputs/Return:** `void`
- **Side effects:** Modifies the global `tmpaasworld` AAS area structure in place.
- **Calls:** Not inferable from this file.
- **Notes:** Part of the AAS build pipeline; must be called after initial area creation but before routing/reachability passes.

### AAS_LadderSubdivision
- **Signature:** `void AAS_LadderSubdivision(void)`
- **Purpose:** Subdivides AAS areas at ladder surface boundaries, ensuring ladder-traversable regions are isolated into their own areas for correct bot movement classification.
- **Inputs:** None (implicitly operates on global `tmpaasworld`)
- **Outputs/Return:** `void`
- **Side effects:** Modifies the global `tmpaasworld` AAS area structure in place.
- **Calls:** Not inferable from this file.
- **Notes:** Ladder areas require special movement handling in the bot AI; subdivision ensures reachability links between ladder and non-ladder areas can be correctly generated.

## Control Flow Notes
Both functions are offline build-time passes invoked during AAS compilation within the BSPC tool. They are not called at game runtime. They slot into the AAS generation pipeline between raw area extraction and the optimization/routing phases (alongside other subdivision passes such as `aas_areamerging` and `aas_edgemelting`).

## External Dependencies
- No includes in this header.
- **Defined elsewhere:** `tmpaasworld` — the global temporary AAS world structure used across the BSPC AAS construction pipeline.
