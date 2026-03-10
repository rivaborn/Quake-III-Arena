# code/bspc/csg.c

## File Purpose
Implements Constructive Solid Geometry (CSG) operations for the BSPC (BSP Compiler) tool. It processes raw map brushes into non-overlapping convex brush sets suitable for BSP tree construction by performing boolean set operations (subtract, intersect, chop) on brush geometry.

## Core Responsibilities
- Validate brush convexity and bounds (`CheckBSPBrush`)
- Generate per-side windings for BSP brushes (`BSPBrushWindings`)
- Merge adjacent compatible brushes into single brushes (`TryMergeBrushes`, `MergeBrushes`)
- Subtract one brush volume from another, splitting as needed (`SubtractBrush`)
- Compute brush intersections and disjoint tests (`IntersectBrush`, `BrushesDisjoint`)
- Carve intersecting brushes into the minimum non-overlapping set (`ChopBrushes`)
- Build the initial brush list from map data, clipped to world bounds (`MakeBspBrushList`)
- Orchestrate the full world brush processing pipeline (`ProcessWorldBrushes`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bspbrush_t` | struct | A convex brush in BSP space with sides, bounds, and a pointer to its source `mapbrush_t` |
| `mapbrush_t` | struct | Original map brush with contents, entity number, and original sides |
| `side_t` | struct | One face of a brush: plane number, winding, texinfo, flags |
| `plane_t` | struct | A map plane defined by normal and distance |
| `tree_t` | struct | BSP tree root with head node and world bounds |
| `node_t` | struct | BSP tree node or leaf, holding brush fragments and portals |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `minplanenums[3]` | `int[3]` | global | Plane indices for clip-box minimum faces (X/Y axes) |
| `maxplanenums[3]` | `int[3]` | global | Plane indices for clip-box maximum faces (X/Y axes) |

## Key Functions / Methods

### CheckBSPBrush
- Signature: `void CheckBSPBrush(bspbrush_t *brush)`
- Purpose: Validates a brush for convexity and in-bounds geometry; logs warnings on failure.
- Inputs: `brush` — brush to validate
- Outputs/Return: void
- Side effects: Calls `BoundBrush` (updates brush bounds); writes to log on error
- Calls: `WindingsNonConvex`, `BoundBrush`, `Log_Print`
- Notes: Checks all side-pair combinations for non-convexity via winding tests.

### BSPBrushWindings
- Signature: `void BSPBrushWindings(bspbrush_t *brush)`
- Purpose: Computes clipped windings for every side of a brush by intersecting with all opposing half-spaces.
- Inputs: `brush` — brush whose sides need windings
- Outputs/Return: void; populates `brush->sides[i].winding`
- Side effects: Allocates winding memory via `BaseWindingForPlane`/`ChopWindingInPlace`
- Calls: `BaseWindingForPlane`, `ChopWindingInPlace`

### TryMergeBrushes
- Signature: `bspbrush_t *TryMergeBrushes(bspbrush_t *brush1, bspbrush_t *brush2)`
- Purpose: Attempts to merge two brushes into one if they share exactly one opposing plane pair and the result remains convex.
- Inputs: Two brushes to merge
- Outputs/Return: New merged `bspbrush_t` on success; `NULL` if merge is impossible
- Side effects: Allocates new brush; calls `BSPBrushWindings`, `BoundBrush`, `CheckBSPBrush`
- Calls: `AllocBrush`, `BSPBrushWindings`, `BoundBrush`, `CheckBSPBrush`, `WindingsNonConvex`, `Log_Print`
- Notes: Requires identical `texinfo` on coplanar sides (unless one is `TEXINFO_NODE`). Does not free input brushes.

### MergeBrushes
- Signature: `bspbrush_t *MergeBrushes(bspbrush_t *brushlist)`
- Purpose: Iteratively merges all compatible brush pairs in a list until no further merges are possible.
- Inputs: Linked list of brushes
- Outputs/Return: Reduced brush list
- Side effects: Frees merged brushes; prints progress via `qprintf`
- Calls: `TryMergeBrushes`, `FreeBrush`, `qprintf`

### SubtractBrush
- Signature: `bspbrush_t *SubtractBrush(bspbrush_t *a, bspbrush_t *b)`
- Purpose: Computes `a - b`; returns list of brush fragments remaining after removing the volume of `b` from `a`.
- Inputs: `a` — brush being carved; `b` — cutter brush
- Outputs/Return: List of result fragments, or original `a` if no intersection
- Side effects: Allocates fragment brushes; frees intermediate brushes
- Calls: `SplitBrush2`, `FreeBrush`, `FreeBrushList`
- Notes: Iterates over each side of `b`, splitting `a` progressively.

### IntersectBrush
- Signature: `bspbrush_t *IntersectBrush(bspbrush_t *a, bspbrush_t *b)`
- Purpose: Returns the volumetric intersection of two brushes, or `NULL` if disjoint.
- Inputs: `a`, `b` — brushes to intersect
- Outputs/Return: Single brush representing `a ∩ b`, or `NULL`
- Side effects: Allocates/frees intermediate brushes
- Calls: `SplitBrush2`, `FreeBrush`

### BrushesDisjoint
- Signature: `qboolean BrushesDisjoint(bspbrush_t *a, bspbrush_t *b)`
- Purpose: Fast conservative disjoint test via AABB and opposing-plane checks.
- Outputs/Return: `true` if definitely disjoint; `false` if possibly intersecting
- Notes: May produce false negatives for non-axial cases.

### ChopBrushes
- Signature: `bspbrush_t *ChopBrushes(bspbrush_t *head)`
- Purpose: Main CSG pass — carves all intersecting brushes into a non-overlapping set using priority-based subtraction.
- Inputs: Linked list of brushes
- Outputs/Return: Non-overlapping brush list
- Side effects: Heavily mutates brush list; respects `cancelconversion` global; prints progress; writes to log
- Calls: `BrushesDisjoint`, `BrushGE`, `SubtractBrush`, `CullList`, `AddBrushListToTail`, `FreeBrushList`, `CountBrushList`, `Log_Print`, `Log_Write`, `qprintf`
- Notes: Uses `goto newlist` for list restart after structural modification. Only accepts subtractions that don't fragment (at least one result is a single piece).

### MakeBspBrushList
- Signature: `bspbrush_t *MakeBspBrushList(int startbrush, int endbrush, vec3_t clipmins, vec3_t clipmaxs)`
- Purpose: Builds a working BSP brush list from the global `mapbrushes` array, copying and clipping each brush to the world bounding box.
- Inputs: Brush index range, clip bounds
- Outputs/Return: Linked list of clipped `bspbrush_t`
- Side effects: Sets `minplanenums`/`maxplanenums` globals; allocates brushes and windings
- Calls: `FindFloatPlane`, `AllocBrush`, `CopyWinding`, `ClipBrushToBox`

### ProcessWorldBrushes
- Signature: `tree_t *ProcessWorldBrushes(int brush_start, int brush_end)`
- Purpose: Top-level pipeline: builds brush list → optionally chops → builds BSP tree.
- Inputs: Brush index range
- Outputs/Return: Completed `tree_t`, or `NULL` on cancellation
- Side effects: Calls `ResetBrushBSP`; reads `map_mins`/`map_maxs` globals; conditionally runs `ChopBrushes` based on `nocsg` flag
- Calls: `MakeBspBrushList`, `ChopBrushes`, `BrushBSP`, `AllocNode`, `Tree_Alloc`, `ResetBrushBSP`, `FreeBrushList`

## Control Flow Notes
This file is part of the **offline map compilation** pipeline (BSPC tool), not the runtime engine. `ProcessWorldBrushes` is the entry point called from higher-level BSP compilation logic. Execution order: `MakeBspBrushList` → `ChopBrushes` (CSG) → `BrushBSP` (tree construction). The `cancelconversion` global allows cooperative cancellation mid-process. The `nocsg` flag bypasses `ChopBrushes`.

## External Dependencies
- `qbsp.h` — all core types (`bspbrush_t`, `mapbrush_t`, `side_t`, `plane_t`, `tree_t`, `node_t`) and shared globals
- `mapplanes[]`, `mapbrushes[]`, `map_mins`, `map_maxs` — defined in `map.c`
- `cancelconversion`, `nocsg`, `create_aas` — defined in `bspc.c`
- `WindingsNonConvex`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding` — defined in winding/polygon utilities
- `AllocBrush`, `FreeBrush`, `CopyBrush`, `SplitBrush`, `BoundBrush`, `BrushBSP`, `CountBrushList`, `FreeBrushList`, `ResetBrushBSP` — defined in `brushbsp.c`
- `FindFloatPlane` — defined in `map.c`
- `AllocNode`, `Tree_Alloc` — defined in `tree.c` / `brushbsp.c`
- `Log_Print`, `Log_Write`, `qprintf` — logging utilities
