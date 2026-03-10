# code/bspc/aas_store.h

## File Purpose
Header for the BSPC (BSP Compiler) tool's AAS storage subsystem, defining capacity limits for all AAS data arrays and declaring the public interface for allocating, freeing, and persisting the compiled AAS world data.

## Core Responsibilities
- Define compile-time maximum element counts for every AAS data structure (bboxes, vertexes, planes, edges, faces, areas, nodes, portals, clusters)
- Expose `aasworld` as the global AAS world state (type `aas_t`)
- Declare the interface to store a finalized AAS world to disk
- Declare helpers for plane lookup and bulk AAS memory management
- Guard against botlib internals being pulled in during BSPC compilation via `BSPCINCLUDE`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `aas_t` | struct (defined in `be_aas_def.h`) | Complete AAS world state: all geometry arrays, routing caches, entity links, and metadata |

> Note: The commented-out `bspc_aas_t` block shows a historical BSPC-local mirror of `aas_t`; it has been superseded by reusing `aas_t` directly.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aasworld` | `aas_t` | global (extern) | Singleton AAS world state shared across all BSPC subsystems; defined in `aas_store.c` |

## Key Functions / Methods

### AAS_StoreFile
- Signature: `void AAS_StoreFile(char *filename)`
- Purpose: Serializes the in-memory `aasworld` to an `.aas` file on disk after BSP compilation.
- Inputs: `filename` — output file path string
- Outputs/Return: void
- Side effects: File I/O; writes the complete AAS binary file
- Calls: Not inferable from this file alone
- Notes: Called at the end of the BSPC compile pipeline once AAS geometry is finalized

### AAS_FindPlane
- Signature: `qboolean AAS_FindPlane(vec3_t normal, float dist, int *planenum)`
- Purpose: Searches the AAS plane table for an existing plane matching the given normal and distance; returns its index if found.
- Inputs: `normal` — plane normal vector; `dist` — plane distance; `planenum` — output index
- Outputs/Return: `qboolean` (true if found); plane index via `*planenum`
- Side effects: None (read-only search, or may insert)
- Calls: Not inferable from this file
- Notes: Used during geometry construction to deduplicate planes

### AAS_AllocMaxAAS
- Signature: `void AAS_AllocMaxAAS(void)`
- Purpose: Pre-allocates all AAS arrays to their maximum capacities (as defined by the `AAS_MAX_*` constants) before geometry is built.
- Inputs: None
- Outputs/Return: void
- Side effects: Heap allocation for all AAS sub-arrays in `aasworld`
- Calls: Not inferable from this file
- Notes: Must be called before any AAS data is written; pairs with `AAS_FreeMaxAAS`

### AAS_FreeMaxAAS
- Signature: `void AAS_FreeMaxAAS(void)`
- Purpose: Releases all AAS arrays allocated by `AAS_AllocMaxAAS`.
- Inputs: None
- Outputs/Return: void
- Side effects: Heap deallocation
- Calls: Not inferable from this file
- Notes: Called on shutdown or after the AAS file has been stored

## Control Flow Notes
This header is part of the **BSPC offline compiler** tool, not the runtime engine. The expected call order is:
1. `AAS_AllocMaxAAS()` — reserve worst-case memory
2. Geometry build passes populate `aasworld` arrays
3. `AAS_StoreFile()` — flush to disk
4. `AAS_FreeMaxAAS()` — release memory

The macro `BSPCINCLUDE` (defined before `#include "be_aas_def.h"`) suppresses the botlib runtime headers (`be_aas_main.h`, routing, entity, etc.) that are irrelevant in the offline tool context.

## External Dependencies
- `../game/be_aas.h` — travel flags, `aas_trace_t`, `aas_entityinfo_t`, `aas_clientmove_t`, and other AAS public types
- `../botlib/be_aas_def.h` — `aas_t` struct definition, all AAS sub-types (`aas_bbox_t`, `aas_area_t`, `aas_reachability_t`, etc.), routing structures
- `vec3_t`, `qboolean` — defined elsewhere in `q_shared.h`
- `aasworld` global — defined in `aas_store.c` (not visible here)
