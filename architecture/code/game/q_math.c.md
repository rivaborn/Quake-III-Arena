# code/game/q_math.c

## File Purpose
Stateless mathematical utility library shared across all Quake III Arena modules (game, cgame, UI, renderer). Provides 3D vector math, angle conversion, plane operations, bounding box utilities, and fast approximation routines.

## Core Responsibilities
- Vector arithmetic: normalize, dot/cross product, rotate, scale, MA operations
- Angle utilities: conversion, normalization, interpolation, delta computation
- Plane operations: construction from points, sign-bit classification, box-plane side testing
- Bounding box management: clear, expand, radius computation
- Direction compression: float normal ↔ quantized byte index via `bytedirs` table
- Fast math approximations: `Q_rsqrt` (Quake fast inverse square root), `Q_fabs`
- Seeded PRNG: `Q_rand`, `Q_random`, `Q_crandom`

## Key Types / Data Structures
None defined in this file; all types come from `q_shared.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `cplane_t` | struct (extern) | Plane with normal, dist, type, signbits — used by `BoxOnPlaneSide` |
| `vec3_t` / `vec4_t` | typedef (extern) | Core float-array vector types |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `vec3_origin` | `vec3_t` | global | Zero vector constant `{0,0,0}` |
| `axisDefault[3]` | `vec3_t[3]` | global | Identity axis matrix |
| `colorBlack`…`colorDkGrey` | `vec4_t` | global | Named RGBA color constants |
| `g_color_table[8]` | `vec4_t[8]` | global | Indexed color table for `^N` chat color codes |
| `bytedirs[NUMVERTEXNORMALS]` | `vec3_t[162]` | global | Icosphere normal lookup table for byte-compressed directions |

## Key Functions / Methods

### Q_rsqrt
- Signature: `float Q_rsqrt(float number)`
- Purpose: Fast reciprocal square root using the famous bit-hack approximation (0x5f3759df), one Newton-Raphson iteration.
- Inputs: `number` — positive float
- Outputs/Return: Approximate `1/sqrt(number)`
- Side effects: None
- Calls: None
- Notes: Disabled on `idppc` (replaced by `frsqrte` asm in header). Contains the notorious "what the fuck?" comment. Single iteration; commented-out second iteration available.

### BoxOnPlaneSide
- Signature: `int BoxOnPlaneSide(vec3_t emins, vec3_t emaxs, struct cplane_s *p)`
- Purpose: Determines which side(s) of a plane an AABB occupies. Returns 1 (front), 2 (back), or 3 (spanning).
- Inputs: Box min/max corners, plane pointer with precomputed `signbits` and `type`
- Outputs/Return: Integer side mask (1, 2, or 3)
- Side effects: None
- Calls: None (pure computation)
- Notes: Two implementations — C (switch on `signbits` 0–7) and x86 MSVC `__declspec(naked)` FPU asm. Linux/FreeBSD x86 uses an external asm path. Axial planes short-circuit before the general case.

### RotatePointAroundVector
- Signature: `void RotatePointAroundVector(vec3_t dst, const vec3_t dir, const vec3_t point, float degrees)`
- Purpose: Rotates `point` around arbitrary axis `dir` by `degrees`.
- Inputs: Destination, axis direction, point, angle in degrees
- Outputs/Return: Result written to `dst`
- Side effects: None
- Calls: `PerpendicularVector`, `CrossProduct`, `MatrixMultiply`, `DEG2RAD`, `cos`, `sin`

### DirToByte / ByteToDir
- Signature: `int DirToByte(vec3_t dir)` / `void ByteToDir(int b, vec3_t dir)`
- Purpose: Compress/decompress a unit normal to/from a 1-byte index into the 162-entry `bytedirs` icosphere table.
- Inputs: Float direction vector or byte index
- Outputs/Return: Best-match index (0–161) / direction copied to `dir`
- Side effects: None
- Calls: `DotProduct`, `VectorCopy`
- Notes: `DirToByte` is O(162) linear scan; not cheap per comment. Returns 0 for null input.

### PlaneFromPoints
- Signature: `qboolean PlaneFromPoints(vec4_t plane, const vec3_t a, const vec3_t b, const vec3_t c)`
- Purpose: Computes plane equation (normal + distance) from three points; normal points out for clockwise winding.
- Inputs: Output plane (xyz=normal, w=dist), three position vectors
- Outputs/Return: `qtrue` on success; `qfalse` if triangle is degenerate
- Side effects: None
- Calls: `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`

### AngleVectors
- Signature: `void AngleVectors(const vec3_t angles, vec3_t forward, vec3_t right, vec3_t up)`
- Purpose: Decomposes Euler angles (pitch/yaw/roll) into three orthogonal basis vectors.
- Inputs: Angles array, output pointers (any may be NULL)
- Outputs/Return: Writes forward, right, up vectors
- Side effects: Uses `static` locals `sr,sp,sy,cr,cp,cy` to work around MSVC FP bugs
- Calls: `sin`, `cos`

### Notes (trivial helpers)
- `ClampChar`/`ClampShort`: Saturate int to signed 8/16-bit range.
- `ColorBytes3`/`ColorBytes4`: Pack float RGB(A) into unsigned int.
- `NormalizeColor`: Scales color vector so max component = 1.
- `AngleMod`/`AngleNormalize360`/`AngleNormalize180`/`AngleDelta`/`LerpAngle`/`AngleSubtract`/`AnglesSubtract`: Angle normalization and interpolation family.
- `ClearBounds`/`AddPointToBounds`/`RadiusFromBounds`: AABB management.
- `VectorNormalize`/`VectorNormalize2`: Normalize in-place or to separate output, returning length.
- `_DotProduct`, `_VectorMA`, `_VectorSubtract`, etc.: Non-macro fallbacks declared in header.

## Control Flow Notes
This file has no init/frame/shutdown lifecycle. It is a pure utility compilation unit compiled into every module (game VM, cgame VM, UI VM, renderer, tools) via `#include "q_shared.h"`. The `#ifdef __LCC__` guards provide non-inline versions for the QVM bytecode compiler, while `#if !idppc` gates the bit-hack rsqrt for non-PowerPC targets.

## External Dependencies
- **Includes**: `q_shared.h` (all type definitions, macros, inline variants)
- **Defined elsewhere**: `assert`, `sqrt`, `cos`, `sin`, `atan2`, `fabs`, `isnan` from `<math.h>`; `memcpy`, `memset` from `<string.h>`; `VectorNormalize` (called by `PerpendicularVector`, defined later in same file); `PerpendicularVector` (called by `RotatePointAroundVector`, defined later in same file — forward reference resolved at link time within TU)
- **Platform asm paths**: x86 MSVC `__declspec(naked)` `BoxOnPlaneSide`; Linux/FreeBSD i386 uses external asm (excluded via `#if` guard)
