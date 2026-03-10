# code/botlib/be_aas_sample.h

## File Purpose
Public and internal header for AAS (Area Awareness System) spatial sampling and querying operations. It exposes functions for point/bbox/trace queries against AAS geometry, and conditionally exposes internal link management functions when compiled with `AASINTERN` defined.

## Core Responsibilities
- Declare presence-type bounding box queries
- Declare point-in-area and area cluster/presence lookups
- Declare client bounding-box trace operations against AAS space
- Declare multi-area trace and bbox overlap enumeration
- Guard internal AAS link heap and entity linking functions behind `AASINTERN`

## Key Types / Data Structures
None defined here; types are declared elsewhere and used by reference.

| Name | Kind | Purpose |
|---|---|---|
| `aas_face_t` | struct (defined elsewhere) | Represents a face in AAS geometry |
| `aas_plane_t` | struct (defined elsewhere) | Plane equation for an AAS face |
| `aas_link_t` | struct (defined elsewhere) | Node in the linked list of areas an entity overlaps |
| `aas_trace_t` | struct (defined elsewhere) | Result of a client bbox trace through AAS space |
| `aas_areainfo_t` | struct (defined elsewhere) | Output container for generic area information |

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_PresenceTypeBoundingBox
- Signature: `void AAS_PresenceTypeBoundingBox(int presencetype, vec3_t mins, vec3_t maxs)`
- Purpose: Returns the mins/maxs of the collision bounding box for a given presence type (standing, crouching, etc.)
- Inputs: `presencetype` — enum-like int; `mins`/`maxs` — output vectors
- Outputs/Return: Filled `mins` and `maxs` via out-params
- Side effects: None inferable
- Calls: Not inferable from this file
- Notes: Presence types gate which areas/traces are valid for a bot posture

### AAS_AreaCluster
- Signature: `int AAS_AreaCluster(int areanum)`
- Purpose: Returns the cluster index an area belongs to; negative value indicates a portal area
- Inputs: `areanum` — AAS area index
- Outputs/Return: Cluster number (positive) or negative portal number
- Side effects: None inferable

### AAS_TraceClientBBox
- Signature: `aas_trace_t AAS_TraceClientBBox(vec3_t start, vec3_t end, int presencetype, int passent)`
- Purpose: Sweeps a client-shaped bounding box from `start` to `end` through AAS space, returning trace result
- Inputs: `start`/`end` — world-space endpoints; `presencetype` — posture; `passent` — entity number to ignore
- Outputs/Return: `aas_trace_t` by value containing hit fraction, end position, face/area info
- Side effects: None inferable
- Notes: Primary movement collision query for bot path validation

### AAS_TraceAreas
- Signature: `int AAS_TraceAreas(vec3_t start, vec3_t end, int *areas, vec3_t *points, int maxareas)`
- Purpose: Records all AAS areas crossed by a line trace, up to `maxareas`
- Inputs: `start`/`end` — trace endpoints; `areas` — output area index array; `points` — output contact points; `maxareas` — array capacity
- Outputs/Return: Count of areas the trace passed through
- Side effects: Writes into caller-supplied `areas` and `points` arrays

### AAS_BBoxAreas
- Signature: `int AAS_BBoxAreas(vec3_t absmins, vec3_t absmaxs, int *areas, int maxareas)`
- Purpose: Enumerates all AAS areas overlapping a given axis-aligned bounding box
- Inputs: `absmins`/`absmaxs` — world-space bbox; `areas` — output buffer; `maxareas` — capacity
- Outputs/Return: Number of overlapping areas written

### AAS_PointAreaNum
- Signature: `int AAS_PointAreaNum(vec3_t point)`
- Purpose: Returns the AAS area index containing the given world-space point
- Inputs: `point` — 3D world coordinate
- Outputs/Return: Area number, or 0 if outside all areas

### Internal (`AASINTERN`) functions
- `AAS_InitAASLinkHeap` / `AAS_FreeAASLinkHeap` — allocate/free the pool of `aas_link_t` nodes
- `AAS_InitAASLinkedEntities` / `AAS_FreeAASLinkedEntities` — init/free per-entity area link lists
- `AAS_AASLinkEntity` / `AAS_LinkEntityClientBBox` — insert an entity into the AAS spatial index
- `AAS_UnlinkFromAreas` — remove an entity from all areas it is linked into
- `AAS_AreaGroundFace`, `AAS_TraceEndFace`, `AAS_PlaneFromNum`, `AAS_PointInsideFace`, `AAS_InsideFace` — low-level geometry helpers

## Control Flow Notes
This header is consumed by `be_aas_sample.c` (which defines `AASINTERN` to access internal declarations) and by any botlib subsystem needing spatial queries (route planning, movement, goal evaluation). It plays no direct role in frame/render loops; it is queried on demand during bot thinking.

## External Dependencies
- `be_aas_def.h` or equivalent (defines `aas_face_t`, `aas_plane_t`, `aas_link_t`, `aas_trace_t`, `aas_areainfo_t`)
- `q_shared.h` (defines `vec3_t`, `qboolean`)
- All function bodies defined in `code/botlib/be_aas_sample.c`
