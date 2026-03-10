# code/botlib/aasfile.h

## File Purpose
Defines the binary file format for AAS (Area Awareness System) data used by the Quake III bot navigation system. It declares all constants, flags, and on-disk data structures that describe how navigable areas, connectivity (reachability), and spatial partitioning are stored in `.aas` files.

## Core Responsibilities
- Define the AAS file magic identifier and version constants
- Enumerate all travel types bots can use to move between areas
- Define face, area content, and area flag bitmasks
- Declare the 14 lump layout for the AAS binary file format
- Provide all geometric and topological structs (vertices, edges, faces, areas, nodes)
- Define cluster/portal structures for hierarchical routing
- Declare the file header struct that indexes all lumps

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_bbox_t` | struct | Bounding box with presence type and flags for bot collision classification |
| `aas_reachability_t` | struct | Directed edge between two areas: travel type, start/end points, travel time |
| `aas_areasettings_t` | struct | Per-area metadata: contents, flags, cluster membership, reachability index range |
| `aas_portal_t` | struct | Cluster portal area linking front/back clusters |
| `aas_cluster_t` | struct | Navigation cluster grouping areas for hierarchical pathfinding |
| `aas_plane_t` | struct | BSP plane with normal, distance, and type |
| `aas_edge_t` | struct | Two-vertex edge by index |
| `aas_face_t` | struct | Polygonal face bounding an area; may separate two areas |
| `aas_area_t` | struct | Convex area with face list, AABB, and center point |
| `aas_node_t` | struct | BSP tree node with plane and two children (negative = area leaf, zero = solid) |
| `aas_lump_t` | struct | File offset and length for a single lump |
| `aas_header_t` | struct | AAS file header: ident, version, BSP checksum, 14 lump descriptors |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header defining constants and data structures only.

## Control Flow Notes
This file is a passive format definition. It is consumed at load time by `be_aas_file.c` (botlib) and by the `bspc` map compiler tool when reading or writing `.aas` files. The lump constants (`AASLUMP_*`) index directly into `aas_header_t::lumps[]`, driving the deserialization sequence. The travel type constants feed into routing logic in `be_aas_route.c` and movement logic in `be_aas_move.c`.

## External Dependencies
- No explicit includes in this header itself
- Depends on `vec3_t` being defined by the including translation unit (from `q_shared.h`)
- Structs are consumed by: `be_aas_file.c`, `be_aas_route.c`, `be_aas_move.c`, `be_aas_reach.c`, `be_aas_cluster.c`, and `bspc/` tool sources
