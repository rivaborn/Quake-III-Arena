# code/bspc/l_math.h

## File Purpose
Defines the core 3D math types, constants, and vector operation interfaces used throughout the BSPC (BSP Compiler) tool. It provides both macro-based inline operations and function declarations for vector arithmetic, normalization, bounds tracking, and rotation utilities.

## Core Responsibilities
- Declare scalar and vector typedefs (`vec_t`, `vec3_t`, `vec4_t`) with optional double-precision via `DOUBLEVEC_T`
- Provide inline macro implementations of common vector ops (dot product, add, subtract, scale, etc.)
- Declare function-form equivalents of vector ops for use where macros are inappropriate
- Declare geometric utilities: normalization, cross product, length, color normalization
- Declare spatial bounds management (`ClearBounds`, `AddPointToBounds`)
- Declare angle/rotation matrix utilities (`AngleVectors`, `R_ConcatRotations`, `CreateRotationMatrix`, `RotatePoint`)
- Define BSP-specific plane side constants (`SIDE_FRONT`, `SIDE_BACK`, `SIDE_ON`, `SIDE_CROSS`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vec_t` | typedef | Scalar float (or double if `DOUBLEVEC_T` defined) |
| `vec3_t` | typedef | 3-component vector array (`vec_t[3]`) |
| `vec4_t` | typedef | 4-component vector array (`vec_t[4]`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vec3_origin` | `vec3_t` | global (extern) | Zero vector constant; defined in `l_math.c` |

## Key Functions / Methods

### VectorCompare
- Signature: `qboolean VectorCompare(vec3_t v1, vec3_t v2)`
- Purpose: Equality test for two vectors within `EQUAL_EPSILON` (0.001) tolerance
- Inputs: Two `vec3_t` vectors
- Outputs/Return: `qboolean` true if components are within epsilon
- Side effects: None
- Calls: Not inferable from this file
- Notes: Uses epsilon, not exact comparison

### VectorNormalize / VectorNormalize2
- Signature: `vec_t VectorNormalize(vec3_t inout)` / `vec_t VectorNormalize2(const vec3_t v, vec3_t out)`
- Purpose: Normalize vector to unit length; first form is in-place, second writes to separate output
- Inputs: Source vector(s)
- Outputs/Return: Original length as `vec_t`
- Side effects: Modifies vector in place (`VectorNormalize`) or `out` (`VectorNormalize2`)
- Calls: Not inferable from this file

### CrossProduct
- Signature: `void CrossProduct(const vec3_t v1, const vec3_t v2, vec3_t cross)`
- Purpose: Computes the cross product of two vectors; result is perpendicular to both
- Inputs: `v1`, `v2` — input vectors
- Outputs/Return: `cross` — result vector
- Side effects: Writes to `cross`
- Calls: Not inferable from this file

### AngleVectors
- Signature: `void AngleVectors(const vec3_t angles, vec3_t forward, vec3_t right, vec3_t up)`
- Purpose: Decomposes Euler angles (pitch/yaw/roll) into an orthonormal basis (forward, right, up)
- Inputs: `angles` in degrees
- Outputs/Return: Fills `forward`, `right`, `up` vectors
- Side effects: Writes to three output vectors
- Calls: Not inferable from this file

### CreateRotationMatrix / RotatePoint / R_ConcatRotations
- Purpose: Build a 3×3 rotation matrix from Euler angles, apply it to a point, and concatenate two rotation matrices, respectively
- Inputs/Outputs: `float[3][3]` matrices, `vec3_t` point
- Notes: Used for brush/geometry rotation during BSP compile

### AddPointToBounds / ClearBounds
- Purpose: Maintain an AABB; `ClearBounds` initializes mins/maxs to extreme values, `AddPointToBounds` expands them to include a point
- Notes: Trivial utility pair, heavily used in spatial partitioning

### ColorNormalize
- Signature: `vec_t ColorNormalize(vec3_t in, vec3_t out)`
- Purpose: Scales a color vector so its largest component equals 1.0
- Inputs: `in` — raw color
- Outputs/Return: `out` — normalized color; returns original max component
- Side effects: Writes `out`

### Q_rint
- Signature: `vec_t Q_rint(vec_t in)`
- Purpose: Round-to-nearest-integer for `vec_t`, avoiding direct `rint` dependency
- Notes: Trivial helper

## Control Flow Notes
This is a header-only interface file. It is included by BSPC source files that perform geometry processing (brush splitting, AAS area creation, etc.). There is no frame or update loop here; these utilities are called during offline map compilation, not at runtime.

## External Dependencies
- `<math.h>` — for `M_PI`, trigonometric functions used by implementations
- `qboolean` — defined elsewhere in BSPC/shared headers (not defined here)
- Implementations: `l_math.c` (defined elsewhere)
