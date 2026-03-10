# q3radiant/Winding.h — Enhanced Analysis

## Architectural Role

This header declares the **core geometric winding manipulation API** for the q3radiant level editor—a pure offline/tool-time module entirely outside the runtime engine. Windings (vertex-ordered polygons representing BSP face geometry) are the fundamental representation that the editor uses when constructing brushes, performing CSG operations, and validating face topology. These functions bridge high-level brush editing operations (in CSG, brush manipulation, and face selection tools) down to low-level geometric primitives, forming the mechanical core of the editor's geometry pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant brush/CSG operations** (`Brush.cpp`, `CSG.CPP`, `MAP.CPP`): all face construction, merging, and boolean operations rely on winding clipping and splitting
- **q3radiant face/brush editing tools** (`SELECT.CPP`, face dialogs): query and modify windings during interactive editing
- **q3radiant map persistence** (`MAP.CPP`): serialize/deserialize brush geometry via windings

### Outgoing (what this file depends on)
- **q3radiant memory management** (likely `StdAfx.h`, `cmdlib.h`): `Winding_Alloc` / `Winding_Free` call into editor heap
- **q3radiant math utilities** (`MATHLIB.H/CPP`): vector cross-products, plane distance calculations, epsilon-based comparisons
- **q3shared types** (`q_shared.h`): `plane_t`, `vec3_t`, `qboolean` definitions
- **common tool library** (`common/mathlib.c`, `common/polylib.c`): may reuse shared polygon clipping logic across q3map and q3radiant

## Design Patterns & Rationale

- **Functional decomposition**: Each geometric operation (clip, split, merge, test) is a discrete, testable function rather than class methods—typical of pre-OOP C codebases and tool-era idiomatic Q3 code
- **In-place vs. allocation**: Functions like `Winding_Clip` and `Winding_InsertPoint` return newly allocated results, while `Winding_RemovePoint` modifies in-place—reflects the tradeoff between safety (no aliasing bugs) and efficiency (avoid thrashing during CSG)
- **Epsilon-based comparisons**: `Point_Equal(epsilon)`, `Winding_SplitEpsilon(epsilon)` hardcode numerical robustness for the offline pipeline—editor geometry is inherently prone to floating-point error from user input and iterative transformations
- **Structural simplicity**: No callbacks, no virtual methods, no state machines—just pure geometric algorithms, enabling safe inlining and compiler optimization in tool code where performance is less critical than correctness

## Data Flow Through This File

1. **Input**: Editor user creates/modifies a brush → CSG code assembles source windings from brush planes
2. **Transformation**: `Winding_Clip`, `Winding_Split`, `Winding_TryMerge` are called in sequence during CSG operations to materialize the result geometry
3. **Validation**: `Winding_IsHuge`, `Winding_PlanesConcave`, `Winding_PointInside` verify topological correctness and warn on degenerate faces
4. **Output**: Resulting windings are serialized back to `.map` file format or cached in editor document for display/further editing

## Learning Notes

- **Tool vs. Runtime split**: Unlike the runtime collision system (`qcommon/cm_*.c`), which optimizes for lookup speed and maintains state, these editor functions are **stateless and allocate freely**—tools can afford higher memory churn for simplicity
- **Geometric robustness idiom (pre-modern)**: The explicit `epsilon` parameters and `qboolean keepon` flags reflect 1990s-era numerical robustness practices; modern engines use interval arithmetic or exact predicates, but Q3 editors standardized on tolerance-based comparisons throughout the offline pipeline
- **Shared offline codebase**: Functions like `Winding_Clip` are conceptually similar to runtime BSP clipping (in bspc or q3map), but this header is editor-specific; the actual implementations differ in memory model and error handling philosophy
- **No integration with AAS/botlib**: The winding system is purely geometric; it has **zero interaction** with the runtime bot navigation (AAS) pipeline—AAS operates on compiled BSP clusters, not editor brushes

## Potential Issues

- **No bounds checking**: `Winding_RemovePoint(w, point)` does not validate `point` index; out-of-bounds access would corrupt the winding. Inferred as acceptable in tool code where the editor UI enforces invariants before calling.
- **Epsilon hardcoding**: Functions accept epsilon as parameter but don't document canonical tolerance values (e.g., is it 0.01 units, 0.1?). Callers must maintain consistency across CSG phases.
- **Winding inversion ambiguity**: `Winding_Reverse` flips vertex order but does not update plane normals; callers must track plane separately (see `Winding_Plane`). Subtle coupling that could cause face-culling inversions if misused.
