# code/bspc/aas_map.h

## File Purpose
Public interface header for the AAS map brush creation module within the BSPC (BSP Compiler) tool. It exposes a single function used to convert BSP map brushes into AAS-compatible geometry.

## Core Responsibilities
- Declares the interface for converting `mapbrush_t` geometry into AAS brush data
- Acts as the include boundary between `aas_map.c` and other BSPC modules that need to create AAS map brushes

## Key Types / Data Structures
None defined in this file. Types `mapbrush_t` and `entity_t` are defined elsewhere (likely `qbsp.h` / `map.h`).

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_CreateMapBrushes
- **Signature:** `void AAS_CreateMapBrushes(mapbrush_t *brush, entity_t *mapent, int addbevels);`
- **Purpose:** Creates AAS brush geometry from a BSP map brush, associating it with its source map entity. The `addbevels` flag controls whether bevel planes are added to the brush for smoother bot movement collision.
- **Inputs:**
  - `brush` — pointer to the source BSP map brush to convert
  - `mapent` — pointer to the map entity owning this brush
  - `addbevels` — boolean-style int; non-zero causes bevel planes to be appended
- **Outputs/Return:** `void`; results are written into AAS internal data structures (side effect)
- **Side effects:** Allocates and populates AAS brush/plane data in the global AAS world being constructed
- **Calls:** Defined in `aas_map.c`; callers are in the BSPC map-to-AAS pipeline (e.g., `aas_create.c`)
- **Notes:** Bevel planes are critical for correct bot navigation; skipping them may cause movement prediction artifacts along brush edges

## Control Flow Notes
This header is consumed during the BSPC offline compilation phase, not at game runtime. It is part of the map-to-AAS conversion pipeline: BSP map data is loaded → brushes are iterated → `AAS_CreateMapBrushes` is called per brush → AAS area/reachability data is built from the results.

## External Dependencies
- `mapbrush_t` — defined in `qbsp.h` or `map.h` (BSPC map representation)
- `entity_t` — defined in BSP entity headers (BSPC)
- Implementation: `code/bspc/aas_map.c`
