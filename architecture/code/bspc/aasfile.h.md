# code/bspc/aasfile.h

## File Purpose
Defines the binary file format for Area Awareness System (AAS) files used by the bot navigation system in Quake III Arena. It specifies all data structures, constants, and lump identifiers needed to read and write `.aas` files that describe navigable regions of a map.

## Core Responsibilities
- Define the AAS binary file layout (header + 14 lumps)
- Enumerate all travel types bots can use to traverse the world
- Define face flags, area contents flags, and area flags for navigation queries
- Declare geometry primitives (vertices, planes, edges, faces, areas, BSP nodes)
- Declare higher-level navigation constructs (reachabilities, area settings, portals, clusters)
- Provide presence type constants for standing vs. crouching bot movement

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_bbox_t` | struct | Bounding box per presence type (normal/crouch) with flags |
| `aas_reachability_t` | struct | Connection between two areas: travel type, start/end points, travel time |
| `aas_areasettings_t` | struct | Per-area metadata: contents, flags, presence type, cluster membership, reachability index |
| `aas_portal_t` | struct | Cluster portal: links two clusters through a shared area |
| `aas_cluster_t` | struct | Group of areas forming a navigation cluster with portal references |
| `aas_vertex_t` | typedef (vec3_t) | 3D point in the AAS geometry |
| `aas_plane_t` | struct | Half-space plane (normal + distance + type) |
| `aas_edge_t` | struct | Pair of vertex indices forming one edge |
| `aas_face_t` | struct | Polygon bounding a convex area; separates two adjacent areas |
| `aas_area_t` | struct | Convex navigable region with AABB and face list |
| `aas_node_t` | struct | BSP tree node; negative child = convex area leaf, zero = solid leaf |
| `aas_lump_t` | struct | Offset + length descriptor for one file lump |
| `aas_header_t` | struct | File header: magic, version, BSP checksum, 14 lump descriptors |
| `aas_edgeindex_t` | typedef (int) | Index into edge array; negative = reversed winding |
| `aas_faceindex_t` | typedef (int) | Index into face array; negative = backside reference |
| `aas_portalindex_t` | typedef (int) | Index into portal array for a cluster |

## Global / File-Static State
None.

## Key Functions / Methods
None — this is a pure header defining constants and data structures only.

## Control Flow Notes
This file is a passive format definition. It is consumed during:
- **Build time**: by `bspc` (the BSP-to-AAS compiler) when writing `.aas` files to disk.
- **Runtime**: by the botlib (`botlib/be_aas_file.c`) when loading `.aas` files and populating the in-memory AAS world representation used every frame for bot pathfinding and routing.

The 14 lump layout (`AASLUMP_*` constants 0–13) mirrors the pattern of Quake BSP file formats. Lump data is referenced sequentially: BSP nodes (`AASLUMP_NODES`) are traversed to locate areas, reachabilities (`AASLUMP_REACHABILITY`) drive A* / routing, and clusters + portals (`AASLUMP_CLUSTERS`, `AASLUMP_PORTALS`) partition the world for hierarchical pathfinding.

## External Dependencies
- `vec3_t` — defined in `q_shared.h` (used for all 3D vectors/vertices)
- No function prototypes; all symbols defined here are data structure and preprocessor constant definitions only.
