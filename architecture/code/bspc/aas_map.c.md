# code/bspc/aas_map.c

## File Purpose
Transforms raw map brushes from BSP entities into AAS-ready geometry during the BSPC (BSP Compiler) offline tool's map conversion process. It handles brush expansion for player bounding boxes, entity validation, content classification, and coordinate-space transformation for moving entities such as rotating doors.

## Core Responsibilities
- Compute signed distances from bounding-box (AABB or capsule) origin offsets relative to brush planes, used for Minkowski-sum expansion
- Expand each `mapbrush_t` outward by a player bounding box so pathfinding geometry accounts for player size
- Set `texinfo` flags on brush sides to control which sides act as BSP splitters
- Validate map entities for AAS relevance (world, func_wall, func_door, triggers, etc.)
- Resolve `trigger_always` activation chains recursively to determine if a rotating door is permanently open
- Transform brush planes into world-space for entities with an `origin` or rotation (func_door_rotating)
- Classify and normalize brush contents (SOLID, LADDER, CLUSTERPORTAL, TELEPORTER, JUMPPAD, MOVER, etc.)
- Duplicate and expand solid/ladder brushes once per configured bounding-box type (normal, crouch)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mapbrush_t` | struct (typedef) | A map brush with sides, contents, bounds, and expansion metadata |
| `side_t` | struct (typedef) | One face of a brush: plane number, texinfo, winding, surface flags |
| `plane_t` | struct (typedef) | Normal + distance plane definition used for all BSP operations |
| `cfg_t` | struct (typedef) | Global AAS config holding all player bounding boxes (`cfg.bboxes[]`, `cfg.numbboxes`) |
| `entity_t` | struct (extern) | Map entity with key-value pairs, origin, and brush list |
| `aas_bbox_t` | struct (typedef) | Bounding box with `presencetype`, `flags`, `mins`, `maxs` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `mark_entities` | `int[MAX_MAP_ENTITIES]` | file-global | Recursion guard for `AAS_AlwaysTriggered_r` cycle detection |
| `cfg` | `cfg_t` (extern) | global (aas_cfg) | AAS configuration including all player bounding boxes |
| `mapplanes` | `plane_t[]` (extern) | global (map.c) | All map planes indexed by planenum |
| `mapbrushes` | `mapbrush_t[]` (extern) | global (map.c) | All loaded map brushes |
| `brushsides` | `side_t[]` (extern) | global (map.c) | All loaded brush sides |
| `nummapbrushes` | `int` (extern) | global (map.c) | Running count of map brushes |
| `nummapbrushsides` | `int` (extern) | global (map.c) | Running count of brush sides |
| `entities` | `entity_t[]` (extern) | global (map.c) | All loaded map entities |
| `num_entities` | `int` (extern) | global (map.c) | Count of loaded entities |
| `capsule_collision` | `qboolean` (extern) | global (bspc.c) | Selects capsule vs. AABB expansion method |

## Key Functions / Methods

### BoxOriginDistanceFromPlane
- **Signature:** `vec_t BoxOriginDistanceFromPlane(vec3_t normal, vec3_t mins, vec3_t maxs, int side)`
- **Purpose:** Returns the signed expansion distance an AABB origin must be offset from a plane so the box just touches it.
- **Inputs:** Plane `normal`; box `mins`/`maxs`; `side` selects which corner (support point) to use.
- **Outputs/Return:** Scalar expansion offset (`vec_t`).
- **Side effects:** None.
- **Calls:** `DotProduct`, `VectorCopy`, `VectorInverse`.
- **Notes:** `side == 0` picks the support point that maximizes overlap (used for expansion); `side != 0` picks the opposite corner.

### CapsuleOriginDistanceFromPlane
- **Signature:** `vec_t CapsuleOriginDistanceFromPlane(vec3_t normal, vec3_t mins, vec3_t maxs)`
- **Purpose:** Computes capsule-specific plane offset based on two sphere centres derived from the box extents.
- **Inputs:** Plane `normal`, box `mins`/`maxs`.
- **Outputs/Return:** Scalar offset for capsule expansion.
- **Side effects:** None.
- **Calls:** None (arithmetic only).
- **Notes:** Only called when `capsule_collision` is true. Uses vertical normal component to select upper or lower sphere.

### AAS_ExpandMapBrush
- **Signature:** `void AAS_ExpandMapBrush(mapbrush_t *brush, vec3_t mins, vec3_t maxs)`
- **Purpose:** Expands every side of a brush outward by the player bounding box (Minkowski sum), registering new float planes.
- **Inputs:** Brush to modify; bounding box mins/maxs.
- **Side effects:** Modifies `s->planenum` for every side; calls `FindFloatPlane` which mutates `mapplanes`/`nummapplanes`; clears `SFL_BEVEL` and `SURF_SKIP` flags.
- **Calls:** `BoxOriginDistanceFromPlane` or `CapsuleOriginDistanceFromPlane`, `FindFloatPlane`.

### AAS_SetTexinfo
- **Signature:** `void AAS_SetTexinfo(mapbrush_t *brush)`
- **Purpose:** Forces `texinfo = 0` (visible splitter) on special-content brushes; sets `TEXINFO_NODE` (skip) on untextured, non-bevel sides of regular brushes.
- **Side effects:** Writes `side->texinfo` for all sides.
- **Calls:** None.

### AAS_AlwaysTriggered_r / AAS_AlwaysTriggered
- **Signature:** `int AAS_AlwaysTriggered_r(char *targetname)` / `int AAS_AlwaysTriggered(char *targetname)`
- **Purpose:** Recursively walks the entity target graph to determine whether a `trigger_always` entity will unconditionally activate a given `targetname` in deathmatch.
- **Inputs:** `targetname` string to check.
- **Outputs/Return:** `true` if always triggered, `false` otherwise.
- **Side effects:** Reads/writes `mark_entities[]` for cycle detection; `AAS_AlwaysTriggered` resets it via `memset`.
- **Calls:** `strlen`, `strcmp`, `ValueForKey`, `atoi`, `Warning`.

### AAS_ValidEntity
- **Signature:** `int AAS_ValidEntity(entity_t *mapent)`
- **Purpose:** Decides whether a map entity contributes geometry to the AAS (world, func_wall, func_door, trigger_hurt, trigger_push, trigger_multiple→teleporter, trigger_teleport, func_static, func_door_rotating always-triggered).
- **Inputs:** Map entity pointer.
- **Outputs/Return:** `true` if entity is AAS-relevant.
- **Calls:** `ValueForKey`, `strcmp`, `atoi`, `AAS_AlwaysTriggered`.

### AAS_TransformPlane
- **Signature:** `int AAS_TransformPlane(int planenum, vec3_t origin, vec3_t angles)`
- **Purpose:** Rotates and translates a plane into world-space for entities with origin/angle offsets.
- **Outputs/Return:** New plane number after registration.
- **Calls:** `VectorCopy`, `CreateRotationMatrix`, `RotatePoint`, `DotProduct`, `FindFloatPlane`.

### AAS_PositionFuncRotatingBrush
- **Signature:** `void AAS_PositionFuncRotatingBrush(entity_t *mapent, mapbrush_t *brush)`
- **Purpose:** Computes the final open/closed angle of a `func_door_rotating` and transforms all brush planes to that position.
- **Side effects:** Modifies all brush side planenums; calls `AAS_MakeBrushWindings`, `AddBrushBevels`, `FreeBrushWindings`.
- **Calls:** `FloatForKey`, `GetVectorForKey`, `VectorMA`, `AAS_TransformPlane`, `AAS_MakeBrushWindings`, `AddBrushBevels`, `FreeBrushWindings`.

### AAS_PositionBrush
- **Signature:** `void AAS_PositionBrush(entity_t *mapent, mapbrush_t *brush)`
- **Purpose:** Translates brush planes by entity origin or delegates to `AAS_PositionFuncRotatingBrush`; sets specialised content flags for trigger_hurt, trigger_push, trigger_multiple, trigger_teleport, func_door.
- **Side effects:** Modifies planenums and `brush->contents`.
- **Calls:** `AAS_PositionFuncRotatingBrush`, `FindFloatPlane`, `DotProduct`, `FloatForKey`, `ValueForKey`.

### AAS_CreateMapBrushes
- **Signature:** `void AAS_CreateMapBrushes(mapbrush_t *brush, entity_t *mapent, int addbevels)`
- **Purpose:** Main entry point called per brush during map loading — validates, positions, normalises contents, optionally bevels, expands per bounding box, and duplicates solid brushes for each configured player bounding box.
- **Side effects:** May reduce `nummapbrushsides`, increment `nummapbrushes`/`mapent->numbrushes`, call `AAS_CopyMapBrush`, `AAS_ExpandMapBrush`, `AAS_MakeBrushWindings`, `AddBrushBevels`, `FreeBrushWindings`.
- **Calls:** `AAS_ValidEntity`, `AAS_PositionBrush`, `AAS_SetTexinfo`, `AAS_MakeBrushWindings`, `AddBrushBevels`, `FreeBrushWindings`, `AAS_CopyMapBrush`, `AAS_ExpandMapBrush`.

### Notes (trivial helpers)
- `AAS_MakeBrushWindings` — builds per-side windings and AABB, validates convexity and bounds; returns `true` always.
- `AAS_FixMapBrush` — similar to `AAS_MakeBrushWindings` but also clamps out-of-range brushes by adding axis-aligned sides; appears unused in this file's main flow (commented-out call).
- `AAS_CopyMapBrush` — shallow-copies a brush and its sides into the global arrays for a given entity.
- `AAS_AddMapBrushSide` — appends one side to a brush used by `AAS_FixMapBrush`.
- `FreeBrushWindings` — releases all winding memory from a brush's sides.

## Control Flow Notes
This file is part of the **offline BSPC tool** (not the runtime engine). It is invoked during `create_aas` map conversion:
1. After each brush is parsed from the BSP/map file, `AAS_CreateMapBrushes` is called.
2. Invalid entities are pruned immediately.
3. Brushes are translated/rotated into world space.
4. Content flags are normalised and classified.
5. For liquid/portal/mover brushes: one copy is expanded by the maximum bbox.
6. For solid/ladder brushes: `cfg.numbboxes` copies are created, one expanded per presence type.
7. Resulting brushes feed into the subsequent BSP → AAS tree construction pipeline.

## External Dependencies
- `qbsp.h` — `mapbrush_t`, `side_t`, `plane_t`, `entity_t`, `mapplanes`, `mapbrushes`, `brushsides`, global counters, winding utilities, `AddBrushBevels`, `FindFloatPlane`
- `l_mem.h` — memory allocation
- `botlib/aasfile.h` — `aas_bbox_t`, presence type constants
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t`, `cfg` global
- `game/surfaceflags.h` — `CONTENTS_*`, `SURF_SKIP`
- **Defined elsewhere:** `ValueForKey`, `FloatForKey`, `GetVectorForKey`, `CreateRotationMatrix`, `RotatePoint`, `VectorMA`, `VectorInverse`, `DotProduct`, `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding`, `WindingsNonConvex`, `ClearBounds`, `AddPointToBounds`, `Log_Print`, `Warning`, `Error`, `memset`
