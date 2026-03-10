# q3map/brush_primit.c — Enhanced Analysis

## Architectural Role

This file provides texture coordinate basis computation for **brush primitive** texture mapping—a more sophisticated texture alignment system introduced in Q3A that replaces simple planar projection. As part of the offline BSP compiler (`q3map/`), it transforms surface normals into orthogonal texture-space vectors during map compilation. The global flag `g_bBrushPrimit` gates the use of this system, allowing mixed legacy and brush-primitive geometry in a single map.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/map.c**, **q3map/brush.c**: during brush/face parsing and validation when `g_bBrushPrimit` is enabled
- **q3radiant/** (editor): parallel implementation in `q3radiant/` must remain byte-identical for editor↔compiler consistency (critical constraint noted in comments)
- **q3map/brush_primit.c callers**: likely surface setup and texture matrix finalization during BSP phase

### Outgoing (what this file depends on)
- **Standard C math**: `atan2()`, `sin()`, `cos()`, `sqrt()`, `fabs()`
- No engine or other subsystem dependencies; purely mathematical utility
- **common/mathlib.h** types: `vec3_t`, `vec_t` (float vector primitives)

## Design Patterns & Rationale

**Axis-from-normal computation via rotation matrices:**
- The function solves the problem: *given a surface normal, find two orthogonal vectors that span the tangent plane*
- Rather than using Gram-Schmidt or other orthogonalization, it decomposes the normal as a **rotation from the X-axis** via two Euler angles (RotY around Y-axis, RotZ around Z-axis)
- Once the rotation is known, it applies that same rotation to the canonical basis vectors `(0,1,0)` and `(0,0,1)` to get `texX` and `texY`
- **Why this way?** Deterministic, easily invertible, and matches the editor's implementation (see comment about `BE THE SAME`)

**Epsilon-cleaning of near-zero components:**
- Lines 37–42 clamp tiny values (< 1e-6) to exact 0.0
- Prevents numerical instability in `atan2` when arguments are supposed to be on axes but have floating-point rounding errors

**Special atan2 case handling:**
- The comment warns that `atan2(y,x)` vs `atan(y/x)` may differ when x==0 depending on platform
- Q3A solves this by always using `atan2`, which is well-defined on all platforms

## Data Flow Through This File

**Input:** A surface normal vector (unit or non-unit; function doesn't normalize)

**Transformation:**
1. Clean tiny components to zero
2. Decompose normal as rotation in spherical coordinates: `RotY = -atan2(normal.z, √(normal.x² + normal.y²))` and `RotZ = atan2(normal.y, normal.x)`
3. Rotate basis vectors `(0,1,0)` and `(0,0,1)` by these rotations
4. Store rotated vectors in `texX` and `texY` (tangent and bitangent in modern graphics terminology)

**Output:** Two orthogonal texture coordinate vectors forming a coordinate frame on the surface

**Used during:** BSP compilation when brush primitive texturing is enabled; the generated frame is baked into the surface data structure for the renderer to use during `tr_shade.c` texture coordinate generation.

## Learning Notes

- **Brush Primitives vs. Legacy:** Q3A supports both old-style planar texture projection (still default) and brush primitives (more artist-friendly alignment). This file is the mathematical heart of the latter.
- **Editor–Compiler coherence:** The repeated warning about matching editor code reflects a real maintenance hazard: if the editor and compiler diverge, maps will light/texture differently between WYSIWYG editing and final compile.
- **No runtime dependency:** Unlike the botlib AAS or renderer systems, this is pure offline tooling—no runtime renderer overhead.
- **Deterministic for QA:** Using explicit matrix rotation (not Gram-Schmidt or other "flexible" methods) ensures bit-identical output across platforms and compilations, important for map validation.

## Potential Issues

- **No input validation:** The function does not check if the input normal is zero-length or near-degenerate. A zero or very small normal would produce undefined `atan2` results or axis-aligned texture vectors pointing in arbitrary directions.
- **Platform-dependent floating-point:** Despite the epsilon cleanup, `sin()/cos()` of `atan2()` results may have minor IEEE 754 rounding differences across architectures, though unlikely to be perceptible in map compilation.
- **Undocumented inverse:** No function to recover the normal from `(texX, texY)` is provided here, though the transformation is mathematically reversible.
