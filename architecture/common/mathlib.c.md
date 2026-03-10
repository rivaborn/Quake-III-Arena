# common/mathlib.c

## File Purpose
Implements the shared 3D math primitive library used by Quake III Arena's offline tools (q3map, bspc, q3radiant, etc.). Provides vector, plane, matrix, and spatial utility operations used across the common tool infrastructure.

## Core Responsibilities
- Vector arithmetic: add, subtract, scale, negate, copy, dot product, cross product
- Vector normalization and length computation
- Plane construction from points and plane type classification
- Bounding box management (clear and expand)
- 3x3 matrix multiplication and point rotation around an arbitrary axis
- Normal-to-compact-encoding conversion (lat/long byte encoding)
- Color vector normalization (max-component scale)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vec_t` | typedef (float or double) | Scalar component type, switchable via `DOUBLEVEC_T` |
| `vec3_t` | typedef (vec_t[3]) | 3-component vector |
| `vec4_t` | typedef (vec_t[4]) | 4-component vector (used for planes: xyz + dist) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vec3_origin` | `vec3_t` | global | Shared zero-vector constant `{0,0,0}` |

## Key Functions / Methods

### NormalToLatLong
- **Signature:** `void NormalToLatLong( const vec3_t normal, byte bytes[2] )`
- **Purpose:** Encodes a unit normal into two bytes (latitude/longitude) for compact storage in lightmap or vertex data.
- **Inputs:** `normal` — unit direction vector; `bytes[2]` — output buffer
- **Outputs/Return:** Writes `bytes[0]` = longitude, `bytes[1]` = latitude
- **Side effects:** None
- **Calls:** `atan2`, `acos`, `RAD2DEG`
- **Notes:** Handles polar singularities (normal aligned to ±Z) as special cases.

### PlaneFromPoints
- **Signature:** `qboolean PlaneFromPoints( vec4_t plane, const vec3_t a, const vec3_t b, const vec3_t c )`
- **Purpose:** Constructs a plane (normal + distance) from three points using the cross product of edge vectors.
- **Inputs:** `a`, `b`, `c` — three points in clockwise winding order
- **Outputs/Return:** Returns `qtrue` on success; fills `plane[0..3]`; returns `qfalse` if degenerate (zero-area triangle)
- **Side effects:** None
- **Calls:** `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`

### VectorNormalize
- **Signature:** `vec_t VectorNormalize( const vec3_t in, vec3_t out )`
- **Purpose:** Normalizes a vector, writing the unit result to `out` and returning the original length.
- **Inputs:** `in` — source vector; `out` — destination (may alias `in`)
- **Outputs/Return:** Returns original length; zero-length input clears `out` and returns 0
- **Side effects:** None
- **Calls:** `sqrt`, `VectorClear`

### RotatePointAroundVector
- **Signature:** `void RotatePointAroundVector( vec3_t dst, const vec3_t dir, const vec3_t point, float degrees )`
- **Purpose:** Rotates `point` around an arbitrary axis `dir` by `degrees`.
- **Side effects:** None
- **Calls:** `PerpendicularVector`, `CrossProduct`, `MatrixMultiply`, `cos`, `sin`, `DEG2RAD`
- **Notes:** Constructs a basis frame aligned to `dir`, builds Z-rotation, then transforms back via inverse (transpose) matrix.

### MatrixMultiply
- **Signature:** `void MatrixMultiply(float in1[3][3], float in2[3][3], float out[3][3])`
- **Purpose:** Standard 3×3 matrix multiply; `out` must not alias `in1` or `in2`.

### ColorNormalize
- **Signature:** `vec_t ColorNormalize( const vec3_t in, vec3_t out )`
- **Purpose:** Scales a color vector so the largest channel equals 1.0; returns the original max value. Zero input yields white (1,1,1).

### Notes
- `_DotProduct`, `_VectorSubtract`, `_VectorAdd`, `_VectorCopy`, `_VectorScale` are function-form fallbacks for the macro versions in `mathlib.h`; used when macro expansion is unsafe (e.g., side-effecting arguments).
- `Vec10Copy` copies a 10-element `vec_t` array; used for extended vertex data (likely lightmap ST + normals).
- `ClearBounds`/`AddPointToBounds` implement a simple AABB accumulation pattern with sentinel values ±99999.
- `PlaneTypeForNormal` returns `PLANE_X/Y/Z` for axis-aligned normals, `PLANE_NON_AXIAL` otherwise, for BSP optimization.

## Control Flow Notes
This file is a stateless utility library. It has no init, frame, or shutdown involvement. It is compiled into offline build tools (`q3map`, `bspc`, `q3radiant`) via the `common/` shared directory. It is **not** part of the runtime game or renderer.

## External Dependencies
- `#include "cmdlib.h"` — provides `qboolean`, `byte`, standard C library headers (`stdio`, `stdlib`, `string`, `math`)
- `#include "mathlib.h"` — self-header; defines macros (`DotProduct`, `VectorSubtract`, etc.), type aliases, and `PLANE_*` constants
- `#pragma optimize("p", on)` — Windows-only: enables floating-point consistency optimization to avoid cross-platform precision divergence in tool computations
- All math functions (`sqrt`, `cos`, `sin`, `atan2`, `acos`, `fabs`, `floor`) come from `<math.h>` via `mathlib.h`
