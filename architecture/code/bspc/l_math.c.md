# code/bspc/l_math.c

## File Purpose
Provides a general-purpose 3D math primitive library for the BSPC (BSP Compiler) tool. Implements vector, rotation, and bounding-box operations used throughout the BSP compilation pipeline. This is a standalone math utility layer, not connected to the runtime game engine.

## Core Responsibilities
- Euler angle decomposition into orthonormal basis vectors (forward/right/up)
- 3×3 rotation matrix concatenation and identity initialization
- Vector arithmetic: add, subtract, scale, dot product, cross product, MA, copy
- Vector normalization (in-place and out-of-place variants)
- Bounding-box management (clear, expand, radius)
- Color channel normalization to [0,1] range

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `vec_t` | typedef (`float` or `double`) | Scalar element of vectors; compile-time switchable via `DOUBLEVEC_T` |
| `vec3_t` | typedef (`vec_t[3]`) | 3-component vector, the primary spatial type |
| `vec4_t` | typedef (`vec_t[4]`) | 4-component vector (declared in header, unused in this file) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vec3_origin` | `vec3_t` | global | Shared zero-vector constant `{0,0,0}` |

Static locals in `AngleVectors` (`sr, sp, sy, cr, cp, cy`) are function-scoped statics used to work around an MSVC floating-point compiler bug; they carry no persistent semantic state across calls.

## Key Functions / Methods

### AngleVectors
- **Signature:** `void AngleVectors(const vec3_t angles, vec3_t forward, vec3_t right, vec3_t up)`
- **Purpose:** Decomposes a Euler angle triple (pitch/yaw/roll, degrees) into three orthogonal basis vectors.
- **Inputs:** `angles` — yaw/pitch/roll in degrees; `forward`, `right`, `up` — output pointers (any may be NULL).
- **Outputs/Return:** Writes to non-NULL output vectors; no return value.
- **Side effects:** None beyond output writes.
- **Calls:** `sin`, `cos`.
- **Notes:** Outputs NULL-guarded individually. Static sin/cos locals are a documented MSVC workaround.

### RadiusFromBounds
- **Signature:** `float RadiusFromBounds(const vec3_t mins, const vec3_t maxs)`
- **Purpose:** Computes the bounding-sphere radius for an AABB by finding the farthest corner from the origin.
- **Inputs:** `mins`, `maxs` — AABB extents.
- **Outputs/Return:** `float` — sphere radius.
- **Side effects:** None.
- **Calls:** `fabs`, `VectorLength`.

### R_ConcatRotations
- **Signature:** `void R_ConcatRotations(float in1[3][3], float in2[3][3], float out[3][3])`
- **Purpose:** Multiplies two 3×3 rotation matrices (`out = in1 * in2`).
- **Inputs:** `in1`, `in2` — source matrices.
- **Outputs/Return:** `out` — product matrix (must not alias inputs).
- **Side effects:** None.
- **Calls:** (none; inline arithmetic only).

### VectorNormalize
- **Signature:** `vec_t VectorNormalize(vec3_t inout)`
- **Purpose:** Normalizes a vector in-place; returns original length.
- **Inputs:** `inout` — vector to normalize.
- **Outputs/Return:** Original length as `vec_t`; writes unit vector back to `inout`. Returns 0 and clears vector if length is 0.
- **Side effects:** Modifies `inout`.
- **Calls:** `sqrt`, `VectorClear`.

### VectorNormalize2
- **Signature:** `vec_t VectorNormalize2(const vec3_t in, vec3_t out)`
- **Purpose:** Non-destructive normalization; writes unit vector to separate output.
- **Inputs:** `in` — source vector; `out` — destination.
- **Outputs/Return:** Original length as `vec_t`.
- **Calls:** `sqrt`, `VectorClear`.

### ColorNormalize
- **Signature:** `vec_t ColorNormalize(vec3_t in, vec3_t out)`
- **Purpose:** Scales an RGB color vector so its brightest channel equals 1.0.
- **Inputs:** `in` — raw RGB; `out` — normalized RGB.
- **Outputs/Return:** Max channel value (pre-normalization); returns 0 if all channels are 0.
- **Calls:** `VectorScale`.

### ClearBounds / AddPointToBounds
- **Notes:** Trivial AABB helpers. `ClearBounds` initializes mins/maxs to sentinel extremes (±99999). `AddPointToBounds` expands the box to include a new point.

### AxisClear
- **Signature:** `void AxisClear(vec3_t axis[3])`
- **Purpose:** Sets a 3×3 axis array to the identity matrix.
- **Side effects:** Writes directly to `axis`.

### Underscore-prefixed vector functions (`_VectorMA`, `_DotProduct`, `_VectorSubtract`, etc.)
- Function-form fallbacks for the macros defined in `l_math.h` (e.g. `DotProduct`, `VectorMA`). Used where macro expansion is unsafe or undesirable.

## Control Flow Notes
This file is a pure utility library with no init/shutdown/frame lifecycle. Functions are called on demand by BSPC geometry processing routines (BSP splitting, AAS generation, etc.). No engine callbacks or event registration.

## External Dependencies
- `l_cmd.h` — pulls in `<stdio.h>`, `<stdlib.h>`, `qboolean` typedef; provides general BSPC utility declarations.
- `l_math.h` — declares all types, macros, and function prototypes implemented here.
- `<math.h>` (via `l_math.h`) — `sin`, `cos`, `sqrt`, `fabs`, `floor`.
- `VectorLength`, `VectorClear`, `VectorScale`, `DotProduct` — defined/macro-expanded within this translation unit or its headers; no external linkage required.
