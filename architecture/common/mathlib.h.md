# common/mathlib.h

## File Purpose
Header-only math library defining 3D vector types, constants, and utility function declarations for use across Quake III Arena's tools (BSP compiler, map tools, bot utilities). It provides the foundational linear algebra primitives shared by tool-side code, distinct from the runtime `q_shared.h` math used in-game.

## Core Responsibilities
- Define scalar and vector types (`vec_t`, `vec2_t`, `vec3_t`, `vec4_t`) with optional double precision
- Declare BSP-relevant plane side constants and plane type classification
- Provide fast inline vector operations via macros (`DotProduct`, `VectorAdd`, etc.)
- Declare function prototypes for non-trivial math operations (normalization, cross product, bounds)
- Declare plane construction and normal encoding utilities
- Declare point rotation utility

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vec_t` | typedef (float or double) | Base scalar type; conditionally double via `DOUBLEVEC_T` |
| `vec2_t` | typedef (array of 3 `vec_t`) | Declared as size 3 — likely a copy-paste artifact; nominally a 2D type |
| `vec3_t` | typedef (`vec_t[3]`) | Primary 3D vector/point type used throughout |
| `vec4_t` | typedef (`vec_t[4]`) | 4-component vector; used to store planes (normal + dist) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vec3_origin` | `vec3_t` | global (extern) | Zero vector constant; defined in `mathlib.c` |

## Key Functions / Methods

### VectorCompare
- Signature: `qboolean VectorCompare( const vec3_t v1, const vec3_t v2 )`
- Purpose: Equality test between two vectors within `EQUAL_EPSILON` (0.001)
- Inputs: Two const vec3_t vectors
- Outputs/Return: `qboolean` true if approximately equal
- Side effects: None
- Calls: Not inferable from header
- Notes: Uses epsilon tolerance, not exact comparison

### VectorNormalize
- Signature: `vec_t VectorNormalize( const vec3_t in, vec3_t out )`
- Purpose: Normalizes `in` to unit length, writes result to `out`
- Inputs: Source vector `in`, destination `out`
- Outputs/Return: Original length of `in`
- Side effects: Writes to `out`
- Calls: Not inferable from header
- Notes: Non-destructive (separate in/out); returns pre-normalization magnitude

### VectorMA
- Signature: `void VectorMA( const vec3_t va, double scale, const vec3_t vb, vec3_t vc )`
- Purpose: Multiply-add: `vc = va + scale * vb`
- Inputs: Base vector `va`, scalar `scale`, direction `vb`, output `vc`
- Outputs/Return: Void; result in `vc`
- Side effects: Writes to `vc`
- Calls: Not inferable from header

### PlaneFromPoints
- Signature: `qboolean PlaneFromPoints( vec4_t plane, const vec3_t a, const vec3_t b, const vec3_t c )`
- Purpose: Constructs a plane (normal + distance) from 3 points using cross product
- Inputs: Three coplanar points `a`, `b`, `c`
- Outputs/Return: `qboolean` false if points are degenerate (collinear); plane written into `plane`
- Side effects: Writes to `plane`
- Calls: Likely `CrossProduct`, `VectorNormalize`

### PlaneTypeForNormal
- Signature: `int PlaneTypeForNormal( vec3_t normal )`
- Purpose: Returns `PLANE_X/Y/Z` if normal is axial, else `PLANE_NON_AXIAL`
- Inputs: A normal vector
- Outputs/Return: Plane type constant (0–3)
- Side effects: None
- Notes: Used in BSP to fast-classify planes for clipping optimizations

### NormalToLatLong
- Signature: `void NormalToLatLong( const vec3_t normal, byte bytes[2] )`
- Purpose: Encodes a unit normal into 2 bytes of latitude/longitude for compact storage (used in BSP lightmap normals)
- Inputs: Unit normal vector
- Outputs/Return: 2-byte encoded output
- Side effects: Writes to `bytes`

### RotatePointAroundVector
- Signature: `void RotatePointAroundVector( vec3_t dst, const vec3_t dir, const vec3_t point, float degrees )`
- Purpose: Rotates `point` around arbitrary axis `dir` by `degrees`
- Inputs: Axis `dir`, source `point`, rotation angle in degrees
- Outputs/Return: Result written to `dst`
- Side effects: Writes to `dst`

**Notes on minor functions:** `Q_rint` rounds a float to nearest integer. `ColorNormalize` normalizes an RGB color vector, clamping to unit scale. `ClearBounds`/`AddPointToBounds` maintain an AABB. `VectorInverse` negates in-place. `Vec10Copy` copies a 10-element `vec_t` array (used for extended vertex data).

## Control Flow Notes
This is a pure header; it participates in compilation of tool-side code (`q3map`, `bspc`, `q3radiant`, `common`). It is included at compile time and has no runtime frame/update role. It is not used by the game runtime, which instead uses the `q_shared.h` math declarations.

## External Dependencies
- `<math.h>` — standard C math (used by implementation in `mathlib.c`)
- `qboolean`, `byte` — defined elsewhere (likely `cmdlib.h` or a shared `q_shared.h` equivalent for tools)
- `vec3_origin` — defined in `common/mathlib.c`
