# q3radiant/brush_primit.cpp — Enhanced Analysis

## Architectural Role

This file implements the **texture matrix mathematics** for Radiant's brush primitive (advanced) texture coordinate system. Brush primitives represent textures via a 2×3 transformation matrix rather than the legacy shift/scale/rotation tuple, enabling precise world-space texture locking during brush editing (move, rotate). The code bridges editor manipulations (face moves/rotations) to deterministic texture updates that must remain **identical to q3map's offline implementation** for map compilation fidelity.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant brush editing UI**: `Face_MoveTexture_BrushPrimit`, `RotateFaceTexture_BrushPrimit`, `Select_ShiftTexture_BrushPrimit` called during interactive brush translate/rotate operations
- **Map/brush parsing**: `BrushPrimit_Parse` invoked during `.map` file load when brushes contain primitive texture blocks
- **Brush face construction**: `FaceToBrushPrimitFace` during initial face setup or format migration
- **Winding/vertex setup**: `EmitBrushPrimitTextureCoordinates` to compute per-vertex texture coordinates from matrix for rendering in editor viewports

### Outgoing (what this file depends on)
- **Texture metadata**: reads `qtexture_t::width`, `qtexture_t::height` (from renderer's texture cache) to scale matrices across resolution changes
- **Vector/math utilities**: `VectorCopy`, `VectorAdd`, `VectorScale`, `DotProduct` from `q_shared.c`/`q_math.c`
- **Winding/Face types**: `face_t`, `winding_t`, `plane_t` from brush/face data model (`qe3.h`)
- **Parser infrastructure**: `GetToken`, `Warning`, `ParseEpair` for map file I/O
- **Editor globals**: `g_qeglobals.bNeedConvert` debug flag for texture conversion validation

## Design Patterns & Rationale

1. **Determinant-based linear solver**: Uses `SarrusDet` (3×3 Sarrus determinant formula) to solve the texture matrix update equations during translate/rotate. This avoids floating-point issues with Gaussian elimination on 2×2 subproblems while keeping math explicit and auditable.

2. **Axis-base coordinate system**: `ComputeAxisBase` derives orthonormal (S,T) vectors from the plane normal via trigonometry—ensuring texture coordinates remain consistent regardless of face orientation. The comments warn about `atan2` sign behavior, reflecting cross-platform portability concerns (macOS/Windows/Linux).

3. **Matrix representation duality**: Functions like `TexMatToFakeTexCoords` / `FakeTexCoordsToTexMat` convert between:
   - **Brush primitive matrix** (raw 2×3 for renderer/q3map)
   - **UI representation** (shift[2], rotation, scale[2])
   
   This allows the editor UI to present familiar shift/scale/rotation controls while the runtime uses the matrix directly.

4. **Texture resolution scaling**: `ConvertTexMatWithQTexture` scales matrix entries by `width_ratio` and `height_ratio` when texture dimensions change. This ensures that swapping a 256×256 texture for a 512×512 doesn't visually stretch the mapping.

5. **Comments signal cross-module contracts**: "must always BE THE SAME as q3map" and TODO comments ("replace texX by texS") indicate this code was manually synchronized with offline tooling, suggesting version control gaps or deliberate compilation-time decoupling of editor and map compiler.

## Data Flow Through This File

**Editing → Texture Lock → Matrix Update:**

1. User drags a brush face or rotates it in the editor viewport.
2. `Select_ShiftTexture_BrushPrimit` or `RotateFaceTexture_BrushPrimit` invoked with delta vector or rotation parameters.
3. Compute plane axis base (`texS`, `texT`) to express motion in local texture space.
4. Solve the matrix equation: find new matrix coefficients that preserve texture world-space positioning.
   - Setup 3×3 system from three reference points (origin, +X, +Y in texture space).
   - Use Sarrus determinant to extract 6 matrix coefficients via Cramer's rule.
5. Update `f->brushprimit_texdef.coords[2][3]` in-place; next render pass uses new matrix.

**Parse → Initialize:**

1. `BrushPrimit_Parse` reads `( ( coords[0][0] ... ) ( coords[1][0] ... ) )` from `.map` file.
2. Also parses old `texdef` metadata (texture name, contents, flags, value) for compatibility.
3. `EmitBrushPrimitTextureCoordinates` later projects winding vertices onto local axes and applies matrix to compute final S/T.

**Legacy Conversion:**

1. `FaceToBrushPrimitFace` migrates old-format faces to new: samples three reference points, emits their S/T via old pipeline, solves for matrix coefficients.
2. Used when importing older `.map` files or format upgrades.

## Learning Notes

1. **Texture Locking as Constraint Satisfaction**: The rotate/move logic models texture locking as a **linear system**—given three fixed reference points in texture space and their new world positions post-transformation, solve for the matrix preserving those mappings. This is elegant and differs from modern PBR engines that often use UV atlases with no-lock semantics.

2. **Determinism across platforms**: The file is meticulous about floating-point stability:
   - Clears near-zero normal components to exact 0 before `atan2`
   - Uses `ZERO_EPSILON` threshold checks
   - Comments warn about `atan2(y, x)` vs. `atan(y/x)` edge cases
   
   Reflects Q3's peer-to-peer server model where brush texture matrices must replicate identically across all machines during gameplay.

3. **Offline/Online Duality**: Comments like "must BE THE SAME as q3map" suggest the designer expected this code to be kept in sync with `code/bspc` and `q3map` by hand. Modern engines unify editor and offline tool codebases; Q3's split reflects late-90s constraints (editor shipped as binary, tools as source).

4. **Matrix Format Stability**: Brush primitive matrices are **baked into `.map` files** (text format), then parsed by both editor and q3map. Any format change requires coordinated updates across tools—hence the cautious comments and validation debug code.

5. **Precision Trade-offs**: The debug check in `EmitBrushPrimitTextureCoordinates` warns when precision loss exceeds `1e-2` (0.01 texture units), indicating tolerance for small errors but intolerance for large divergences. Practical constraint for coordinate precision in 256-unit texture space.

## Potential Issues

- **Silent format incompatibility**: If `q3map` diverges from editor texture math, maps compile differently than they appear in Radiant—no automated check prevents this (only cross-module code review).
- **Determinant singularity**: If brush geometry degenerates (e.g., coplanar points), `det` approaches zero; division in `Face_MoveTexture_BrushPrimit` line ~353 could produce NaN. No zero-det guard is present.
- **Texture resolution assumptions**: `ConvertTexMatWithQTexture` defaults to 2×2 texture size if `qtex2 == NULL`, but if actual texture is different, the scaling is incorrect. The "basic 2×2" assumption should be documented or validated.
